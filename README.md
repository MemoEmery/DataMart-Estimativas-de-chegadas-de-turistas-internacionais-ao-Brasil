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

## 📖 Dicionário de Dados (Data Dictionary)

Esta seção detalha o significado de cada tabela, coluna, chaves de relacionamento e as regras de negócio aplicadas às métricas do Data Mart.

---

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
É a tabela central do modelo estrela. Ela não armazena textos descritivos, apenas as chaves numéricas que apontam para os contextos (dimensões) e os fatos numéricos (métricas).

| Nome da Coluna | Tipo de Dado | Tipo de Chave | Descrição / Significado |
| :--- | :--- | :--- | :--- |
| `id_fato` | SERIAL | PK (Primary Key) | Identificador exclusivo de cada linha de fato registrada no banco. |
| `id_localidade` | INTEGER | FK (Foreign Key) | Chave estrangeira que se conecta à tabela `dim_localidade`. Determina *quem* está vindo e *para onde* vai. |
| `id_via_acesso` | INTEGER | FK (Foreign Key) | Chave estrangeira que se conecta à tabela `dim_via_acesso`. Determina *como* o turista chegou. |
| `id_tempo` | INTEGER | FK (Foreign Key) | Chave estrangeira que se conecta à tabela `dim_tempo`. Determina *quando* a viagem ocorreu. |

#### 📊 Métricas e Regras de Negócio (Campos Quantitativos)

*   ### `quantidade_chegadas` *(Tipo: INTEGER)*
    *   **O que significa:** É a métrica base do projeto. Representa a estimativa calculada do volume físico de indivíduos com nacionalidade estrangeira que cruzaram a fronteira brasileira e iniciaram uma estadia no país dentro do mês e ano referenciados.
    *   **Regra de Agregação:** Esta métrica é **totalmente aditiva**. Isso significa que ela pode ser somada com segurança em qualquer dimensão (Ex: Somar por Ano, somar por Via de Acesso, somar por País de Origem) para gerar indicadores consolidados e KPIs de turismo.

---

