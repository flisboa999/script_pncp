## 📄 README: PNCP Data Miner (Pipeline de Contratos Públicos)

Este projeto implementa um *pipeline* de dados (ETL simplificado) em Python para automatizar a coleta, persistência e classificação inteligente de licitações públicas extraídas diretamente da API do Portal Nacional de Contratações Públicas (PNCP).

O principal objetivo é utilizar Machine Learning (ML) para automatizar a triagem de milhares de registros, permitindo que analistas ou pesquisadores foquem apenas nos contratos classificados como "Relevantes".

### 1. Arquitetura e Etapas Concisas

O *script* segue uma arquitetura baseada em três etapas primárias, focando na aplicação de Machine Learning para otimizar a filtragem de dados públicos:

| Etapa | Ação Principal | Tecnologias Chave | Detalhamento Técnico |
| :--- | :--- | :--- | :--- |
| **1. Coleta** | Extração e Persistência de Dados Brutos. | `requests`, `sqlite3` | Gerenciamento da **Paginação da API** e armazenamento JSON persistente e indexado. |
| **2. Treinamento** | Aprendizado de Padrões de Relevância. | `scikit-learn`, `joblib` | Criação de um modelo para reconhecer padrões em textos de objeto. |
| **3. Classificação** | Filtragem Inteligente dos Dados. | `joblib`, `pandas` | Uso do modelo treinado para classificar automaticamente cada licitação como "Relevante" (Classe 1) ou "Irrelevante" (Classe 0). |

---

### 2. Detalhamento Técnico da Coleta e Persistência

#### 2.1. Interação com a API: Paginação e Resiliência

O acesso aos dados da API do PNCP é feito de forma paginada (ex: 50 registros por página):

* **Paginação Automática:** O script interroga a API, identifica o número total de páginas (`totalPages`) na resposta e utiliza um *loop* controlado para incrementar o parâmetro `pagina` na URL, garantindo que todo o volume de dados seja extraído.
* **Mecanismo de *Retry*:** Para manter a estabilidade da coleta, a função `fetch_com_retry` emprega um **backoff exponencial** (`time.sleep()`). Este mecanismo aumenta o tempo de espera entre tentativas após falhas de conexão ou limites de taxa (*Rate Limiting*), tornando a coleta resiliente e mais eficaz.

#### 2.2. Persistência de Dados e Indexação JSON (SQLite/JSON1)

* **Armazenamento Bruto:** Os dados são armazenados no banco de dados local **SQLite** (`pncp_data.db`) em uma coluna primária (`dados_json`) que preserva o **JSON completo e íntegro** recebido da API.
* **Otimização com JSON1:** Para acelerar consultas, a extensão **JSON1** do SQLite é utilizada para criar colunas virtuais/geradas (*Generated Columns*) a partir de chaves específicas do JSON (ex: `objeto_compra`). Essa técnica permite que índices tradicionais SQL sejam aplicados diretamente nos metadados do JSON, otimizando drasticamente o desempenho de busca sem a necessidade de *parsing* manual do JSON a cada consulta.

---

### 3. Machine Learning (Classificação de Texto)

A aplicação de ML transforma o *pipeline* de um simples extrator de dados para uma ferramenta de inteligência de negócios:

1.  **Pré-Processamento (NLP):** O campo de texto do objeto da licitação é limpo (remoção de pontuação, *stopwords* e tokenização).
2.  **Vetorização (TF-IDF):** O `TfidfVectorizer` (Term Frequency-Inverse Document Frequency) transforma os textos em vetores numéricos. O TF-IDF pesa as palavras, destacando termos que são estatisticamente mais relevantes para o contexto da classificação.
3.  **Treinamento (SGDClassifier):** Um `SGDClassifier` (Stochastic Gradient Descent) é treinado para mapear os vetores TF-IDF para as classes de relevância. O modelo é persistido como `modelo_relevancia.joblib`.
4.  **Benefício:** O modelo reduz o tempo de análise manual, classificando automaticamente a relevância de novos documentos em segundos.

---

### 4. Pontos Fortes e Fracos do Script

#### ✅ Pontos Fortes

* **Otimização de Query (JSON1 Indexing):** Uso avançado do SQLite para criar índices em chaves internas do JSON, garantindo consultas de metadados rápidas e eficientes.
* **Resiliência na Coleta:** O *backoff exponencial* na requisição assegura que a coleta de grandes volumes de dados seja concluída de forma confiável.
* **Filtragem Inteligente (ML):** A aplicação de Machine Learning permite uma triagem automatizada e focada, entregando um produto final com alto valor agregado para o analista.

#### ⚠️ Pontos Fracos e Melhorias Necessárias

* **ERRO CRÍTICO NO TREINAMENTO (IMEDIATO):**
    * O *output* da célula de treinamento (Célula 3) gerou um erro (`The number of classes has to be greater than one; got 1 class`).
    * **Causa Técnica:** O `dataset.csv` usado está **desbalanceado**, contendo apenas exemplos da classe '1' (Relevante). O classificador precisa de pelo menos duas classes (Relevante e Irrelevante) para aprender.
    * **Ação Necessária:** **É indispensável** incluir exemplos de licitações irrelevantes (Classe 0) no `dataset.csv` para corrigir este erro e viabilizar a classificação precisa.
* **Parâmetros *Hardcoded*:** O filtro por `CODIGO_MODALIDADE = 6` (Pregão Eletrônico) está fixo no código.
    * *Melhoria:* Converter parâmetros chave de busca para argumentos de função ou variáveis de configuração.
* **Escalabilidade:** O uso de SQLite é ideal para prototipagem e uso local, mas não é recomendado para ambientes de produção.
    * *Melhoria:* Sugere-se migrar para PostgreSQL ou MySQL em uma fase de produção.