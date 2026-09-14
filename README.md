# Semana_15
Mini Projeto
## 🎯 Perguntas de Negócio (Business Questions)

**Visão Macro e Financeira (Overview)**
* Qual é o volume financeiro total (R$) movimentado nas aquisições de saúde durante o período analisado?
* Qual é a modalidade de compra (ex: Pregão, Inexigibilidade, Dispensa) que movimenta o maior volume financeiro e qual possui o maior número de transações?
* Qual é a proporção do orçamento público investida na categoria de Medicamentos Genéricos em comparação aos medicamentos de referência (marca) e outros materiais?
* Entre os Tipos de Compra classificados no sistema, qual é o mais frequente e qual consome a maior parte dos recursos?

**Análise de Produtos (Curva ABC)**
* Quais são os produtos específicos (categorias/descrição) que compõem a Curva A, ou seja, a minoria de itens que consome 80% do orçamento total?
* Qual é o produto ou categoria que apresenta o maior gasto médio por aquisição, indicando alto custo unitário ou alto volume por pedido?

**Análise de Fornecedores e Risco de Mercado**
* O volume financeiro está concentrado em poucos fornecedores? (Indicador de dependência de mercado ou possível risco de monopólio no fornecimento de insumos críticos).
* Quais são os top 5 fornecedores que mais receberam recursos públicos no período?

**Distribuição Geográfica e Institucional**
* Quais estados (UF) e municípios concentram o maior volume financeiro absoluto em compras?
* Existe um padrão regional nas licitações? Qual é a modalidade de compra mais utilizada por cada estado?

**Análise Temporal e Sazonalidade (2020-2026)**
* Como se comportou a evolução dos gastos totais e da quantidade de itens comprados ano a ano entre 2020 e 2026?
* Existe sazonalidade nas compras públicas de saúde? (Ex: Há meses ou trimestres específicos onde as requisições e empenhos historicamente aumentam?)

## 🛠️ Processamento e Limpeza de Dados (ETL)

O processo de preparação dos dados consistiu na consolidação de bases anuais (2020 a 2026) e no tratamento de inconsistências, valores nulos e regras de negócio. Abaixo estão documentadas as etapas realizadas utilizando Python e a biblioteca `pandas`.

### 1. Extração e Consolidação (Data Ingestion)
* **Leitura Múltipla:** Leitura dos arquivos CSV anuais (2020 a 2026) garantindo a delimitação correta com `sep=';'`.
* **Empilhamento:** Consolidação de todos os anos em um único DataFrame através do `pd.concat`, criando a base unificada para análise histórica.

### 2. Limpeza Básica e Estrutural
* **Remoção de Duplicatas:** Identificação e exclusão de linhas perfeitamente duplicadas (cliques duplos de sistema ou erros de exportação), representando apenas 0.01% da base.
* **Remoção de Nulos Marginais:** Exclusão de linhas com valores nulos em colunas estruturais onde a ausência representava menos de 1% da base (`unidade_fornecimento` e `unidade_fornecimento_capacidade`).

### 3. Recuperação e Imputação de Dados (Data Imputation)
* **Proxy de Datas:** Para valores nulos na coluna `insercao` (data de inserção no sistema), utilizamos os dados da coluna `compra` (data da compra) como proxy temporal.
* **Mapeamento de CNPJ (De/Para):** Para preencher valores nulos em `nome_instituicao`, foi criado um dicionário dinâmico mapeando os nomes das instituições através dos registros existentes na coluna `cnpj_instituicao`. As linhas que permaneceram nulas e sem correspondência de CNPJ foram descartadas.

### 4. Tratamento de Regras de Negócio (Saúde/Compras)
As colunas específicas do domínio de saúde receberam tratamentos para não prejudicar futuras análises estatísticas e classificações:
* **`generico`:** A coluna foi padronizada (letras maiúsculas e sem espaços). Valores vazios (itens que não são medicamentos) foram preenchidos com **`NA`** (Não Aplicável).
* **`anvisa`:** Os nulos nesta coluna foram preenchidos com a tag **`SEM_REGISTRO`**.

### 5. Extração de Features com Regex (Feature Engineering)
Para não inviabilizar a análise numérica da coluna `capacidade` (que continha muitos nulos), aplicamos técnicas de processamento de texto:
* **Padrão de Busca:** Foi elaborada uma expressão regular (`Regex`) para ler a coluna `descricao_catmat` e extrair informações embutidas no texto, como números e unidades (ex: de "DIPIRONA 500 MG", extraiu-se `500` e `MG`).
* **Preenchimento:** Os valores nulos da coluna numérica `capacidade` foram substituídos pelos números extraídos, após conversão de formato (troca de vírgula por ponto). Valores nulos da coluna `unidade_medida` foram substituídos pelos textos extraídos.
* **Tratamento de Resíduos:** Após a extração, os itens que efetivamente não possuíam capacidade tiveram a `capacidade` preenchida com `0` (mantendo o tipo *float* da coluna) e a `unidade_medida` como **`SEM_MEDIDA`**.

### 6. Modelagem de Dados (Star Schema)
Para otimizar a performance dos cálculos no dashboard e garantir as melhores práticas de Business Intelligence, o arquivo consolidado foi transformado em um Modelo Estrela (*Star Schema*).

O modelo separa os atributos descritivos das métricas quantitativas, resultando na seguinte estrutura:
* **Tabelas de Dimensão (Descritivas):** Receberam chaves substitutas numéricas (*Surrogate Keys* - ex: `id_produto`, `id_instituicao`) para otimizar os relacionamentos. Foram geradas:
  * `Dim_Instituicao`: Informações sobre os compradores (Nome, Esfera, UF, Município).
  * `Dim_Fornecedor`: Informações sobre os vendedores (CNPJ, Razão Social).
  * `Dim_Fabricante`: Informações sobre as marcas fabricantes.
  * `Dim_Produto`: Características técnicas dos itens (Código, Descrição, Genérico, Anvisa, Capacidade, Unidade de Medida).
  * `Dim_Tempo`: Calendário completo gerado a partir das datas do projeto (Ano, Mês, Dia, Trimestre), utilizando o padrão numérico `YYYYMMDD` como chave.
* **Tabela Fato (Métricas):** A `Fato_Compras` centraliza os eventos numéricos (`qtd_itens_comprados`, `preco_unitario`, `preco_total`), armazenando apenas os IDs das dimensões associadas. Utilizamos *Role-Playing Dimensions* ao incluir duas chaves de tempo na Fato (`id_tempo_compra` e `id_tempo_insercao`) para permitir análises temporais distintas utilizando o mesmo calendário.

### 7. Controle de Versão e Gestão de Arquivos Grandes
Devido às restrições de tamanho de arquivo do GitHub (limite de 100 MB), os arquivos de dados brutos originais e as tabelas finais exportadas não foram versionados no repositório.

* **Arquivos Ignorados (`.gitignore`):** Todos os arquivos `.csv` presentes nos diretórios de bases anuais (`Dados/brutos`) foram incluídos no `.gitignore`. Foram deixadas apenas as as tabelas exportadas do modelo estrela.
* **Reprodutibilidade:** Para reproduzir o projeto, os scripts Python (`.py` ou `.ipynb`) estão versionados. O usuário precisa apenas baixar os dados brutos originais ([text](https://dadosabertos.saude.gov.br/dataset/bps)), inseri-los no diretório indicado no script e executar o código para que o pipeline de ETL gere automaticamente a base limpa e o modelo estrela localmente.