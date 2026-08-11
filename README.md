# Data Mart de Estimativas de Chegadas de Turistas Internacionais ao Brasil

Este projeto consiste em um Data Mart modelado em arquitetura dimensional (Star Schema / Modelo Estrela) desenvolvido no **PostgreSQL (via pgAdmin 4)**. O objetivo é consolidar e estruturar os dados brutos de estimativas de chegadas de turistas internacionais ao Brasil para facilitar análises analíticas (Business Intelligence e consultas OLAP).

## 📊 Diagrama do Modelo Dimensional

Abaixo está a representação visual das conexões entre a tabela fato e suas respectivas dimensões gerada a partir da ferramenta ERD Tool do pgAdmin 4:

<img width="1153" height="745" alt="Estimativas de chegadas de turistas internacionais ao Brasil- Modelo Dimencional pgerd" src="https://github.com/user-attachments/assets/1cce292a-ea10-48ec-91a8-b1b28f29cad0" />


> 💡 **Nota:** Para exibir a imagem acima, salve o print da tela do seu diagrama na mesma pasta deste arquivo README com o nome exato de `modelo_dimensional.png`.

---

## 📐 Estrutura Arquitetural (Star Schema)

A modelagem foi dividida entre tabelas de contexto (**Dimensões**) e uma tabela centralizadora de métricas numéricas (**Fato**).

### Dicionário de Tabelas

#### 1. dim_localidade
*   `id_localidade` (SERIAL, PK): Identificador único da combinação de origem e destino.
*   `pais_origem` (VARCHAR): País de residência permanente do turista.
*   `uf_destino` (VARCHAR): Estado (Unidade da Federação) por onde o turista entrou/visitou no Brasil.

#### 2. dim_via_acesso
*   `id_via_acesso` (SERIAL, PK): Identificador único do meio de transporte.
*   `descricao_via` (VARCHAR): Tipo de transporte (Ex: Aérea, Terrestre, Marítima, Fluvial).

#### 3. dim_tempo
*   `id_tempo` (SERIAL, PK): Identificador único do período.
*   `ano` (INTEGER): Ano da chegada.
*   `mes` (VARCHAR): Nome do mês correspondente.

#### 4. fato_chegadas
*   `id_fato` (SERIAL, PK): Identificador único do evento.
*   `id_localidade` (INTEGER, FK): Relacionamento com `dim_localidade`.
*   `id_via_acesso` (INTEGER, FK): Relacionamento com `dim_via_acesso`.
*   `id_tempo` (INTEGER, FK): Relacionamento com `dim_tempo`.
*   `quantidade_chegadas` (INTEGER): Métrica contendo a estimativa de turistas que entraram no país.

---

## 🚀 Arquitetura do Processo de Carga (ETL)

1.  **Staging Area:** Os dados brutos foram importados via arquivo CSV gerado em codificação `WIN1252` para manter a integridade dos caracteres acentuados da língua portuguesa para a tabela temporária `staging_turistas_internacionais`.
2.  **Carga das Dimensões:** Utilização de cláusulas `SELECT DISTINCT` para isolar os atributos únicos de texto e povoar as dimensões gerando chaves substitutas (*Surrogate Keys*) automáticas via tipo `SERIAL`.
3.  **Carga da Fato:** Realização de mapeamento via `JOIN` textual entre a tabela de staging e as dimensões criadas, convertendo as colunas de texto brutos em IDs numéricos indexados de alta performance.

---

## 📊 Exemplos de Consultas Analíticas (OLAP)

Com o modelo estrela pronto, você pode executar consultas rápidas de alta performance para responder perguntas de negócio.

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
🔧 **Tecnologias Utilizadas:** PostgreSQL 16+, pgAdmin 4 (ERD Tool).
