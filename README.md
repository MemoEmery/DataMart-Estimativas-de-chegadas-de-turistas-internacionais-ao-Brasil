# Data Mart de Estimativas de Chegadas de Turistas Internacionais ao Brasil

Este projeto consiste em um Data Mart modelado em arquitetura dimensional (Star Schema / Modelo Estrela) desenvolvido no **PostgreSQL (via pgAdmin 4)**. O objetivo é consolidar e estruturar os dados brutos de estimativas de chegadas de turistas internacionais ao Brasil para facilitar análises analíticas (Business Intelligence e consultas OLAP).

## 📊 Diagrama do Modelo Dimensional

Abaixo está a representação visual das conexões entre a tabela fato, suas dimensões e a camada de ingestão isolada (Staging), gerada a partir da ferramenta ERD Tool do pgAdmin 4:

<img width="1150" height="990" alt="DataMart - Estimativas de chegadas de turistas internacionais ao Brasil pgerd" src="https://github.com/user-attachments/assets/1c996d02-07d8-4cbf-91fe-aaa0e19a223d" />

---

## 🏗️ Arquitetura e Ingestão de Dados

A arquitetura do Data Mart foi dividida em duas camadas principais dentro do PostgreSQL:

### 1. Camada de Staging (`staging_turistas_internacionais`)
A tabela de staging atua como **Landing Zone (Área de Pouso)** para a carga dos arquivos brutos em formato CSV (codificação `WIN1252` ou `UTF8`, delimitados por `;`).

* **Estratégia de Ingestão:** Todas as colunas foram modeladas como `VARCHAR(150)` flexível. Essa abordagem garante a integridade na ingestão direta e evita falhas físicas de importação (*type mismatch*, parsing de datas ou inteiros).
* **Tratamento no Pipeline:** Os dados dessa tabela são consumidos pelas transformações no Pentaho Data Integration (PDI) ou scripts SQL, onde são aplicadas as regras de limpeza, conversão de tipos (*CAST*) e mapeamento de chaves substitutas (*Surrogate Keys*).

| Coluna Staging | Tipo | Descrição / Destino no Data Mart |
| :--- | :--- | :--- |
| `continente` | VARCHAR(150) | Nome do continente de origem |
| `cod_continente` | VARCHAR(150) | Código numérico do continente |
| `nome_pais_correto` | VARCHAR(150) | Mapeado para `dim_localidade.pais_origem` |
| `cod_pais` | VARCHAR(150) | Código do país de origem |
| `uf` | VARCHAR(150) | Mapeado para `dim_localidade.uf_destino` |
| `cod_uf` | VARCHAR(150) | Código do estado de destino |
| `via_acesso` | VARCHAR(150) | Mapeado para `dim_via_acesso.descricao_via` |
| `cod_via` | VARCHAR(150) | Código do meio de transporte/acesso |
| `ano` | VARCHAR(150) | Convertido para `INTEGER` em `dim_tempo.ano` |
| `mes` | VARCHAR(150) | Mapeado para `dim_tempo.mes` |
| `cod_mes` | VARCHAR(150) | Código numérico do mês |
| `chegadas` | VARCHAR(150) | Convertido para `INTEGER` em `fato_chegadas.quantidade_chegadas` |

---

### 2. Camada Multidimensional (Star Schema)

A modelagem final do Data Mart foi construída de forma otimizada, removendo colunas técnicas desnecessárias para garantir máxima performance em consultas analíticas.

#### 1. `dim_localidade`
* `id_localidade` (SERIAL, PK): Identificador único da combinação de origem e destino.
* `pais_origem` (VARCHAR): País de residência permanente do turista.
* `uf_destino` (VARCHAR): Estado (Unidade da Federação) por onde o turista entrou/visitou no Brasil.

#### 2. `dim_via_acesso`
* `id_via_acesso` (SERIAL, PK): Identificador único do meio de transporte.
* `descricao_via` (VARCHAR): Tipo de transporte (Ex: Aérea, Terrestre, Marítima, Fluvial).

#### 3. `dim_tempo`
* `id_tempo` (SERIAL, PK): Identificador único do período.
* `ano` (INTEGER): Ano da chegada.
* `mes` (VARCHAR): Nome do mês correspondente.

#### 4. `fato_chegadas`
* `id_localidade` (INTEGER, FK): Relacionamento com `dim_localidade`.
* `id_via_acesso` (INTEGER, FK): Relacionamento com `dim_via_acesso`.
* `id_tempo` (INTEGER, FK): Relacionamento com `dim_tempo`.
* `quantidade_chegadas` (INTEGER): Métrica contendo a estimativa de turistas que entraram no país.

---

## 📖 Dicionário de Dados (Data Dictionary)

### 1. Tabela: `dim_localidade`
Esta dimensão armazena o contexto geográfico da viagem, mapeando o ponto de origem internacional e o ponto de entrada no território brasileiro.

| Nome da Coluna | Tipo de Dado | Tipo de Chave | Descrição / Significado | Exemplo |
| :--- | :--- | :--- | :--- | :--- |
| `id_localidade` | SERIAL | PK (Primary Key) | Chave substituta (*Surrogate Key*) gerada automaticamente para identificar unicamente a combinação de origem e destino. | `1`, `2`, `3` |
| `pais_origem` | VARCHAR(100) | Atributo | O país onde o turista internacional possui residência permanente (origem do fluxo). | `Argentina`, `França` |
| `uf_destino` | VARCHAR(100) | Atributo | A Unidade da Federação (Estado brasileiro) que serviu como porta de entrada principal ou destino registrado do turista. | `PE`, `RJ`, `SP` |

---

### 2. Tabela: `dim_via_acesso`
Esta dimensão categoriza o meio de transporte utilizado pelo turista para cruzar a fronteira e entrar no Brasil.

| Nome da Coluna | Tipo de Dado | Tipo de Chave | Descrição / Significado | Exemplo |
| :--- | :--- | :--- | :--- | :--- |
| `id_via_acesso` | SERIAL | PK (Primary Key) | Chave substituta usada para indexar e identificar unicamente o meio de transporte. | `1`, `2` |
| `descricao_via` | VARCHAR(100) | Atributo | Descrição textual do meio de transporte utilizado na fronteira. | `Aérea`, `Terrestre`, `Marítima`, `Fluvial` |

---

### 3. Tabela: `dim_tempo`
Esta dimensão estabelece o contexto cronológico dos eventos, essencial para análises de sazonalidade, séries temporais e comparações ano a ano.

| Nome da Coluna | Tipo de Dado | Tipo de Chave | Descrição / Significado | Exemplo |
| :--- | :--- | :--- | :--- | :--- |
| `id_tempo` | SERIAL | PK (Primary Key) | Chave substituta para indexação temporal exclusiva deste modelo. | `10`, `11` |
| `mes` | VARCHAR(50) | Atributo | O nome por extenso do mês em que a chegada do turista foi registrada. | `Janeiro`, `Fevereiro` |
| `ano` | INTEGER | Atributo | O ano civil com quatro dígitos correspondente ao registro. | `2023`, `2024` |

---

### 4. Tabela: `fato_chegadas`
É a tabela central do modelo estrela. Ela armazena exclusivamente as chaves numéricas que apontam para as dimensões e a métrica quantitativa do evento.

| Nome da Coluna | Tipo de Dado | Tipo de Chave | Descrição / Significado |
| :--- | :--- | :--- | :--- |
| `id_localidade` | INTEGER | FK (Foreign Key) | Chave estrangeira que se conecta à tabela `dim_localidade`. Determina *quem* está vindo e *para onde* vai. |
| `id_via_acesso` | INTEGER | FK (Foreign Key) | Chave estrangeira que se conecta à tabela `dim_via_acesso`. Determina *como* o turista chegou. |
| `id_tempo` | INTEGER | FK (Foreign Key) | Chave estrangeira que se conecta à tabela `dim_tempo`. Determina *quando* a viagem ocorreu. |
| `quantidade_chegadas` | INTEGER | Métrica | Volume físico de indivíduos com nacionalidade estrangeira que cruzaram a fronteira brasileira. |

#### 📊 Métricas e Regras de Negócio
* **`quantidade_chegadas` (INTEGER):** É a métrica base do projeto. Representa a estimativa calculada do volume de turistas que cruzaram a fronteira e iniciaram estadia no país dentro do mês e ano referenciados.
* **Regra de Agregação:** Métrica **totalmente aditiva**, podendo ser somada de forma simples e segura através de qualquer dimensão (por Ano, por Via de Acesso, por País de Origem).

---

## 📊 Exemplos de Consultas Analíticas (OLAP)

### 1. Total de turistas por Ano e por País de Origem
```sql
SELECT 
    t.ano,
    l.pais_origem,
    SUM(f.quantidade_chegadas) AS total_turistas
FROM fato_chegadas f
JOIN dim_tempo t ON f.id_tempo = t.id_tempo
JOIN dim_localidade l ON f.id_localidade = l.id_localidade
GROUP BY t.ano, l.pais_origem
ORDER BY total_turistas DESC;
```

### 2. Preferência de Via de Acesso por Estado de Destino
```sql
SELECT 
    l.uf_destino,
    v.descricao_via,
    SUM(f.quantidade_chegadas) AS total_turistas
FROM fato_chegadas f
JOIN dim_localidade l ON f.id_localidade = l.id_localidade
JOIN dim_via_acesso v ON f.id_via_acesso = v.id_via_acesso
GROUP BY l.uf_destino, v.descricao_via
ORDER BY l.uf_destino, total_turistas DESC;
```

---

🔧 **Tecnologias Utilizadas:** PostgreSQL 16+, pgAdmin 4 (ERD Tool), Pentaho Data Integration (PDI), Power BI.
