# TrabalhoSAD — Análise e Clusterização do Disque 100 (Colab)

Este repositório/documento descreve o notebook **TrabalhoSAD.ipynb**, que realiza:
1) limpeza e pré-processamento,  
2) transformação categórica (one-hot),  
3) redução de dimensionalidade (PCA) e **clusterização (KMeans)**,  
4) análise dos clusters com métricas (Elbow/ Silhouette),  
5) criação de variáveis agregadas (flags) e **visualizações** (heatmaps, barras interativas, bolhas, Sankey).


## Sumário
- [Visão Geral](#visão-geral)
- [Requisitos](#requisitos)
- [Como Executar](#como-executar)
- [Estrutura do Pipeline](#estrutura-do-pipeline)
  - [1. Leitura e consolidação dos CSVs](#1-leitura-e-consolidação-dos-csvs)
  - [2. Limpeza e pré-processamento](#2-limpeza-e-pré-processamento)
  - [3. Agregação por `hash` (consolidação de registros)](#3-agregação-por-hash-consolidação-de-registros)
  - [4. Matrizes para Clusterização (Padrões e Perfil)](#4-matrizes-para-clusterização-padrões-e-perfil)
  - [5. PCA + KMeans + Validação (Elbow / Silhouette)](#5-pca--kmeans--validação-elbow--silhouette)
  - [6. Flags agregadas e perfis por cluster](#6-flags-agregadas-e-perfis-por-cluster)
  - [7. Gráficos e análises](#7-gráficos-e-análises)
- [Principais DataFrames](#principais-dataframes)
- [Funções principais](#funções-principais)
- [Observações e cuidados](#observações-e-cuidados)
- [Possíveis melhorias](#possíveis-melhorias)

## Visão Geral

O notebook trabalha com dados do Disque 100 e busca identificar **perfis** e **padrões** por meio de clusterização:
- **Padrões de Caso (df_S / X_S)**: canal, denunciante, cenário, frequência, grupo vulnerável, motivação, relação, violação, UF etc.
- **Perfil Vítima/Suspeito (df_P / X_P)**: sexo/gênero, faixa etária, orientação, raça/cor, deficiência, relação etc.

A ideia é gerar clusters interpretáveis e produzir visualizações que facilitem a análise dos grupos.


## Requisitos

### Ambiente
- Google Colab (recomendado) ou Python 3.8+

### Bibliotecas
- pandas, numpy
- matplotlib, seaborn
- scikit-learn
- plotly
- networkx (importado; uso opcional)


## Como Executar

1. Coloque os arquivos `.csv` na mesma pasta do notebook (ou ajuste o `path`).
2. Execute as células na ordem:

   * **Leitura dados** → gera `dataset_disque100.csv`
   * **Pré Processamento** → limpeza e padronização
   * **Agrupamento** → agrega registros por `hash`
   * **Clusterização** → PCA + KMeans (perfil e/ou padrões)
   * **Flags e Gráficos** → interpreta e visualiza clusters

## Estrutura do Pipeline

### 1. Leitura e consolidação dos CSVs

* Busca todos os CSVs via `glob("*.csv")`
* Lê com `sep=";"` e `encoding="latin1"`
* Concatena tudo em um único DataFrame (`df`)
* Re-encoda colunas e strings para `utf-8`
* Salva como `dataset_disque100.csv`

### 2. Limpeza e pré-processamento

Inclui:

* remoção de colunas 100% ausentes
* normalização de nomes das colunas (`clean_col`)
* conversões:

  * `data_de_cadastro`, `inicio_das_violacoes` → datetime
  * `sl_quantidade_vitimas` → numérico
* padronização textual: `.strip().upper()` e vazios viram NA
* mapeamentos binários (SIM/NÃO → 1/0):

  * `denuncia_emergencial`, `suspeito_preso`
* parsing:

  * `pais` (antes do `|`)
  * `municipio` vira `cod_municipio` e `municipio`
  * `violacao` vira `eixo`, `tipo_violacao`, `subtipo_violacao`
* features temporais: `ano`, `mes`
* flags auxiliares: `vitima_com_deficiencia`, `suspeito_com_deficiencia`
* remoção de colunas consideradas ruidosas/irrelevantes para a análise

### 3. Agregação por `hash` (consolidação de registros)

Como um `hash` pode ter várias linhas (mesmo caso), o notebook agrega:

* numéricos: `max` (ex.: `sl_quantidade_vitimas`)
* boolean-like "SIM/NÃO": vira "SIM" se qualquer linha tiver SIM
* categóricos: usa moda (categoria mais frequente)

Resultado: `df_hash`

### 4. Matrizes para Clusterização (Padrões e Perfil)

#### Padrões de caso (`df_S`)

Seleciona colunas mais relacionadas ao evento/caso:

* canal, denunciante, cenário, frequência, grupo vulnerável, motivação, relação, violação, UF etc.

Cria matriz:

* `X_S = prepara_matrix(df_S, num_cols=["sl_quantidade_vitimas"])`

  * categóricos → one-hot com `get_dummies(dummy_na=True)`
  * numéricos → `StandardScaler`

#### Perfil vítima/suspeito (`df_P`)

Seleciona colunas de atributos demográficos e relacionais e cria:

* `X_P = prepara_matrix(df_P, num_cols=[])`

### 5. PCA + KMeans + Validação (Elbow / Silhouette)

Para reduzir dimensionalidade e facilitar clusterização:

* PCA exploratório com `nComponents=15` para avaliar variância explicada
* PCA final com `n_components=0.90` (mantém 90% da variância)
* Elbow Method (WCSS) para sugerir k
* KMeans final com `n_init=50` e `max_iter=1000`
* Silhouette Score médio e silhouette por ponto
* Plot do silhouette + visualização 2D (primeiros 2 componentes)

> No notebook há **dois blocos** desse fluxo:

* um para **perfil vítima/suspeito**
* outro para **padrões de caso**

### 6. Flags agregadas e perfis por cluster

Para facilitar interpretação, o notebook cria **flags** a partir de colunas one-hot:

* vítima: sexo, faixa etária (criança/idoso), deficiência, raça/cor, LGBTQIA+
* suspeito: sexo/nao informado
* relação: familiar/conjugal/institucional/desconhecido

Além disso, cria blocos agregados:

* canal (digital/presencial/telefônico/não informado)
* denunciante (vítima/agressor/terceiro/não info)
* cenário (domicílio/institucional/via pública/virtual)
* frequência (única/ocasional/semanal/diária/não info)
* grupo vulnerável (mulher/criança/idoso/pcd/lgbtq)
* violação macro (física/psíquica/sexual/institucional)
* UF por região (Norte/Nordeste/Centro-Oeste/Sudeste/Sul/desconhecida)

### 7. Gráficos e análises

O notebook gera:

* heatmap de médias (%) por cluster (atributos selecionados)
* barras interativas (plotly) por cluster para cada atributo
* gráfico de bolhas (plotly) para interseções de flags (ex.: vítima x relação)
* heatmap de grupo etário vítima × suspeito por cluster
* heatmaps:
  * cenário × grupo vulnerável por cluster
  * região × grupo vulnerável por cluster
* gráfico de barras empilhadas: composição regional dos clusters
* Sankey: fluxo Cluster → Canal / Cluster → Cenário (plotly)

## Principais DataFrames

* `df`: dataset bruto concatenado
* `df_hash`: dataset consolidado por `hash`
* `df_S`: subset para padrões de caso
* `df_P`: subset para perfil vítima/suspeito
* `X_S`, `X_P`: matrizes one-hot para clusterização
* `X_PCA`, `XS_PCA`: dados transformados pelo PCA
* `dfClusteredB`: matriz (X_P ou X_S) com coluna `Cluster`
* `df_flags`: flags interpretáveis criadas a partir de one-hot
* `dfClusterMeans`: médias por cluster (em proporção, depois convertido para %)
* `out`: DataFrame com clusters + variáveis agregadas (canal, cenário, região etc.)

### Transforma DataFrame em matriz numérica:

* categóricos → one-hot
* numéricos → padroniza com `StandardScaler`
* concatena e preenche NA com 0

### Visualizações (principais)

* `gerar_grafico_barras_interativo(df_pct, atributos, ordenar=False)`
* `plot_bubble_chart_sim_only(df, y_attrs, x_attrs, cluster_col="Cluster")`
* `heatmap_agegroup_vitima_vs_suspeito(df, cluster=..., normalize="row")`
* `sankey_cluster_to_feature(df, cluster_col, feature_cols, title)`
