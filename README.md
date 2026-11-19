# Data Mart de Estimativas de Chegadas de Turistas Internacionais ao Brasil

Data Mart em arquitetura dimensional (Star Schema / Modelo Estrela), desenvolvido no **PostgreSQL** (pgAdmin 4), com carga da staging via **Pentaho Data Integration (PDI)** e visualização em **Power BI**. O objetivo é consolidar as estimativas de chegadas de turistas internacionais ao Brasil (janeiro/2022 a junho/2026) para análises de BI e consultas OLAP.

## Diagrama do Modelo Dimensional

Diagrama gerado pela ERD Tool do pgAdmin 4, com a tabela fato, as quatro dimensões e a camada de staging:

![Diagrama do Data Mart](<img width="1450" height="1303" alt="DataMart - Estimativas de chegadas de turistas internacionais ao Brasil pgerd" src="https://github.com/user-attachments/assets/3b52ed0e-08af-4cc0-bf98-abcd49b21098" />
)

## Fonte dos dados

Cinco arquivos CSV (delimitador `;`, codificação LATIN1/WIN1252), um por ano:

| Arquivo | Período | Linhas | Total de chegadas |
| :--- | :--- | ---: | ---: |
| `chegadas_2022.csv` | 2022 | 54.684 | 3.630.031 |
| `chegadas_2023.csv` | 2023 | 34.764 | 5.908.341 |
| `chegadas_2024.csv` | 2024 | 22.856 | 6.773.619 |
| `chegadas-2025.csv` | 2025 | 24.800 | 9.287.196 |
| `base_chegadas_2026_acum_06_junho.csv` | jan a jun/2026 | 12.859 | 5.261.733 |
| **Total** | | **149.963** | **30.860.920** |

Os arquivos de 2022 a 2024 têm 12 colunas (com códigos e continente). Os de 2025 e 2026 têm só 6 colunas (via, UF, país, mês, ano e chegadas). Por isso os cinco foram consolidados em um único CSV no formato da staging (`staging_turistas_internacionais.csv`, UTF-8), com as colunas ausentes em branco.

## Arquitetura e fluxo de carga

```
CSVs (5 arquivos)
      │  consolidação em 1 CSV (12 colunas, UTF-8)
      ▼
Pentaho (PDI): CSV file input ──► Table output
      ▼
PostgreSQL: staging_turistas_internacionais   (149.963 linhas)
      │  scripts SQL: 04_carga_dim_tempo.sql e 05_correcoes_data_mart.sql
      ▼
dim_tempo · dim_pais_origem · dim_uf_destino · dim_via_acesso
      ▼
fato_chegadas   (72.670 linhas)
      ▼
Power BI (dashboard)
```

| Etapa | Ferramenta | O que faz |
| :--- | :--- | :--- |
| Ingestão | Pentaho (PDI) | Lê o CSV consolidado e grava na staging, com `Truncate table` ligado para não duplicar |
| Dimensão tempo | `04_carga_dim_tempo.sql` | Gera os 60 meses de jan/2022 a dez/2026 com `generate_series` |
| Padronização, dimensões e fato | `05_correcoes_data_mart.sql` | Padroniza textos na staging, recria as dimensões e recarrega a fato |
| Visualização | Power BI | Importa as 5 tabelas do modelo e monta o dashboard |

### 1. Camada de Staging (`staging_turistas_internacionais`)

Área de pouso dos dados brutos. Todas as colunas são `VARCHAR(150)`, o que evita falhas de importação por tipo. As conversões (`CAST`) e o mapeamento de chaves substitutas acontecem nos scripts SQL.

| Coluna da staging | Tipo | Destino no Data Mart |
| :--- | :--- | :--- |
| continente | VARCHAR(150) | `dim_pais_origem.continente` (só existe em 2022 a 2024) |
| cod_continente | VARCHAR(150) | Não utilizado |
| nome_pais_correto | VARCHAR(150) | `dim_pais_origem.pais_origem` |
| cod_pais | VARCHAR(150) | Não utilizado (os códigos mudam entre os anos) |
| uf | VARCHAR(150) | `dim_uf_destino.uf_destino` |
| cod_uf | VARCHAR(150) | Não utilizado |
| via_acesso | VARCHAR(150) | `dim_via_acesso.descricao_via` |
| cod_via | VARCHAR(150) | Não utilizado |
| ano | VARCHAR(150) | `dim_tempo.ano` (convertido para INTEGER) |
| mes | VARCHAR(150) | `dim_tempo.mes` |
| cod_mes | VARCHAR(150) | Não utilizado (só existe em 2022 a 2024) |
| chegadas | VARCHAR(150) | `fato_chegadas.quantidade_chegadas` (convertido para INTEGER) |

### 2. Camada Multidimensional (Star Schema)

| Tabela | Chave | Colunas |
| :--- | :--- | :--- |
| `dim_tempo` | `id_tempo` (SERIAL, PK) | ano, mes, mes_num, mes_abrev, bimestre, trimestre, semestre, ano_mes, data_ref |
| `dim_pais_origem` | `id_pais_origem` (SERIAL, PK) | pais_origem, continente |
| `dim_uf_destino` | `id_uf_destino` (SERIAL, PK) | uf_destino, regiao, detalhada |
| `dim_via_acesso` | `id_via_acesso` (SERIAL, PK) | descricao_via |
| `fato_chegadas` | PK composta: `id_pais_origem`, `id_uf_destino`, `id_via_acesso`, `id_tempo` | quantidade_chegadas |

Contagem de registros após a carga: `dim_tempo` 60, `dim_pais_origem` 212, `dim_uf_destino` 26, `dim_via_acesso` 4 e `fato_chegadas` 72.670.

## Dicionário de Dados

### `dim_tempo`
Granularidade mensal, de janeiro/2022 a dezembro/2026.

| Coluna | Tipo | Chave | Descrição | Exemplo |
| :--- | :--- | :--- | :--- | :--- |
| id_tempo | SERIAL | PK | Chave substituta, em ordem cronológica | 1, 2, 3 |
| ano | INTEGER | Atributo | Ano civil | 2024 |
| mes | VARCHAR(50) | Atributo | Nome do mês por extenso | Março |
| mes_num | SMALLINT | Atributo | Número do mês (usado para ordenar o mês no Power BI) | 3 |
| mes_abrev | CHAR(3) | Atributo | Abreviação do mês | Mar |
| bimestre | SMALLINT | Atributo | Bimestre do ano (1 a 6) | 2 |
| trimestre | SMALLINT | Atributo | Trimestre do ano (1 a 4) | 1 |
| semestre | SMALLINT | Atributo | Semestre do ano (1 ou 2) | 1 |
| ano_mes | CHAR(7) | Atributo | Ano e mês no formato AAAA-MM | 2024-03 |
| data_ref | DATE | Atributo | Primeiro dia do mês (usada como tabela de datas no Power BI) | 2024-03-01 |

Restrição `UNIQUE (ano, mes)`, que impede meses duplicados.

### `dim_pais_origem`
País de residência permanente do turista e seu continente.

| Coluna | Tipo | Chave | Descrição | Exemplo |
| :--- | :--- | :--- | :--- | :--- |
| id_pais_origem | SERIAL | PK | Chave substituta | 1, 2 |
| pais_origem | VARCHAR(100) | Atributo | Nome do país de origem | Argentina |
| continente | VARCHAR(60) | Atributo | Continente de origem | América do Sul |

Os grupos "Outros países" de 2022 a 2024 foram separados por continente (por exemplo, "Outros países - Europa"), porque o mesmo nome aparecia em 5 continentes. O continente dos 114 países que só existem em 2025 e 2026 foi atribuído por mapeamento manual no script `05_correcoes_data_mart.sql`.

### `dim_uf_destino`
Unidade da Federação por onde o turista entrou no Brasil.

| Coluna | Tipo | Chave | Descrição | Exemplo |
| :--- | :--- | :--- | :--- | :--- |
| id_uf_destino | SERIAL | PK | Chave substituta | 1, 2 |
| uf_destino | VARCHAR(100) | Atributo | Nome do estado | Pernambuco |
| regiao | VARCHAR(30) | Atributo | Região do Brasil | Nordeste |
| detalhada | BOOLEAN | Atributo | `FALSE` para "Outras Unidades da Federação", `TRUE` para estados individuais | TRUE |

### `dim_via_acesso`
Meio de transporte usado para entrar no Brasil.

| Coluna | Tipo | Chave | Descrição | Exemplo |
| :--- | :--- | :--- | :--- | :--- |
| id_via_acesso | SERIAL | PK | Chave substituta | 1, 2 |
| descricao_via | VARCHAR(100) | Atributo | Meio de transporte | Aérea, Terrestre, Marítima, Fluvial |

### `fato_chegadas`
Tabela central do modelo estrela, com as chaves das dimensões e a métrica.

| Coluna | Tipo | Chave | Descrição |
| :--- | :--- | :--- | :--- |
| id_pais_origem | INTEGER | PK / FK | Liga a `dim_pais_origem`: de onde vem o turista |
| id_uf_destino | INTEGER | PK / FK | Liga a `dim_uf_destino`: por onde entra |
| id_via_acesso | INTEGER | PK / FK | Liga a `dim_via_acesso`: como chega |
| id_tempo | INTEGER | PK / FK | Liga a `dim_tempo`: quando chega |
| quantidade_chegadas | INTEGER | Métrica | Estimativa de turistas internacionais que chegaram |

## Métricas e regras de negócio

- **`quantidade_chegadas`** é a métrica base: a estimativa de turistas que cruzaram a fronteira no mês e ano referenciados.
- **Regra de agregação:** métrica totalmente aditiva, somada por qualquer dimensão (ano, via, país, estado, continente).
- **Chaves repetidas na fonte:** os CSVs têm várias linhas com a mesma combinação de país, UF, via, ano e mês. A carga da fato usa `SUM` com `GROUP BY`, e os totais por ano batem exatamente com a soma dos CSVs.
- **Linhas com zero chegadas** (55.925 na fonte, quase todas em 2022 e 2023) não são carregadas na fato. Os totais não mudam.

## Qualidade dos dados: tratamentos aplicados

| Problema encontrado | Tratamento |
| :--- | :--- |
| Via de acesso com 6 valores (Aéreo/Aérea e Marítimo/Marítima em 2023) | Padronizada para Aérea e Marítima, resultando em 4 vias |
| Março escrito de três formas (Março, março e Marco) | O join com `dim_tempo` ignora caixa e cedilha |
| "Outros países" repetido em 5 continentes | Renomeado com o continente, por exemplo "Outros países - África" |
| Arquivos de 2025 e 2026 sem continente | Mapeamento manual de 114 países para o continente |
| Coluna de UF com `character(2)` | Alterada para `VARCHAR(100)` |
| Meses duplicados na `dim_tempo` carregada por `DISTINCT` | `dim_tempo` gerada por `generate_series`, com `UNIQUE (ano, mes)` |

### Limitações da fonte (importante para a análise)

- **A granularidade muda em 2025.** Em 2022 a 2024, a UF aparece agrupada em "Outras Unidades da Federação" e há países agrupados em "Outros países...". A partir de 2025 a fonte detalha estados e países. Análises por estado são confiáveis só para 2025 e 2026.
- **2026 vai só até junho.** Para comparar anos, use os mesmos meses (janeiro a junho).
- **Piauí e Tocantins não aparecem em nenhum dos arquivos**, por isso a `dim_uf_destino` tem 26 registros (25 estados e o grupo "Outras Unidades da Federação").
- Os códigos (`cod_pais`, `cod_uf` etc.) mudam entre os anos e só existem até 2024, então os nomes são usados como chave.

## Validação da carga

Total de chegadas por ano na `fato_chegadas`, conferido contra a soma dos CSVs:

| Ano | Total |
| :--- | ---: |
| 2022 | 3.630.031 |
| 2023 | 5.908.341 |
| 2024 | 6.773.619 |
| 2025 | 9.287.196 |
| 2026 (jan a jun) | 5.261.733 |

```sql
SELECT t.ano, SUM(f.quantidade_chegadas) AS total
FROM fato_chegadas f
JOIN dim_tempo t ON t.id_tempo = f.id_tempo
GROUP BY t.ano
ORDER BY t.ano;
```

## Dashboard no Power BI

Conexão: PostgreSQL, modo **Importar**, com as 5 tabelas do modelo (sem a staging). Medida principal:

```
Total Chegadas = SUM('public fato_chegadas'[quantidade_chegadas])
```

Visuais do dashboard:

| Visual | Campos | O que mostra |
| :--- | :--- | :--- |
| Cartão | Total Chegadas | Total de chegadas no período |
| Gráfico de linhas | `ano_mes` e Total Chegadas | Evolução mensal e sazonalidade |
| Segmentação de dados | `ano` | Filtro por ano, que atualiza todos os visuais |
| Colunas | `continente` e Total Chegadas | Chegadas por continente de origem |
| Rosca | `descricao_via` e Total Chegadas | Participação de cada via de acesso |
| Barras | `pais_origem` e Total Chegadas (filtro Top 10) | Principais países de origem |

Configurações do modelo: `mes` classificado por `mes_num`, e `dim_tempo` marcada como tabela de datas, usando `data_ref`.

## Exemplos de consultas analíticas (OLAP)

**1. Total de turistas por ano e por país de origem**
```sql
SELECT
    t.ano,
    p.pais_origem,
    SUM(f.quantidade_chegadas) AS total_turistas
FROM fato_chegadas f
JOIN dim_tempo t        ON f.id_tempo = t.id_tempo
JOIN dim_pais_origem p  ON f.id_pais_origem = p.id_pais_origem
GROUP BY t.ano, p.pais_origem
ORDER BY total_turistas DESC;
```

**2. Preferência de via de acesso por estado de destino (a partir de 2025)**
```sql
SELECT
    u.uf_destino,
    v.descricao_via,
    SUM(f.quantidade_chegadas) AS total_turistas
FROM fato_chegadas f
JOIN dim_uf_destino u   ON f.id_uf_destino = u.id_uf_destino
JOIN dim_via_acesso v   ON f.id_via_acesso = v.id_via_acesso
JOIN dim_tempo t        ON f.id_tempo = t.id_tempo
WHERE t.ano >= 2025
GROUP BY u.uf_destino, v.descricao_via
ORDER BY u.uf_destino, total_turistas DESC;
```

**3. Chegadas por continente e trimestre**
```sql
SELECT
    t.ano,
    t.trimestre,
    p.continente,
    SUM(f.quantidade_chegadas) AS total_turistas
FROM fato_chegadas f
JOIN dim_tempo t        ON f.id_tempo = t.id_tempo
JOIN dim_pais_origem p  ON f.id_pais_origem = p.id_pais_origem
GROUP BY t.ano, t.trimestre, p.continente
ORDER BY t.ano, t.trimestre, total_turistas DESC;
```

## Arquivos do projeto

| Arquivo | Descrição |
| :--- | :--- |
| `04_carga_dim_tempo.sql` | Gera e carrega a `dim_tempo` mensal e recarrega a fato |
| `05_correcoes_data_mart.sql` | Padroniza a staging, recria as dimensões e recarrega a fato |
| `staging_turistas_internacionais.csv` | CSV consolidado (12 colunas, UTF-8) usado na carga da staging |
| `dashboard_turismo.pbix` | Dashboard do Power BI |

## Tecnologias utilizadas

PostgreSQL 16+, pgAdmin 4 (ERD Tool), Pentaho Data Integration (PDI) e Power BI Desktop.
