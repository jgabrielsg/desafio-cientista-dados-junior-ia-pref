# PLAN.md: Auditoria de Classificadores 1746 - Casa Civil / IplanRio

## 1. Escopo e Diretrizes de Avaliação
* Objetivo: Avaliar a substituição do Modelo A (Produção) pelo Modelo B (Candidato) na classificação de chamados do 1746.
* Foco Primário: Profundidade analítica, rigor estatístico e comunicação executiva transparente sobre limitações.

## 2. Decisões Metodológicas e Fundamentação
As diretrizes abaixo embasam as escolhas estatísticas e dão o suporte analítico para a execução do projeto:

* **Métricas para Desbalanceamento:** Será dada preferência ao F1-Score (macro e ponderado) em detrimento da acurácia global. 
    * Justificativa: A acurácia é uma métrica frágil em datasets desbalanceados, pois pode mascarar uma performance ruim em classes minoritárias se o modelo classificar corretamente a classe majoritária. O F1-Score Macro trata todas as classes com o mesmo peso, penalizando falhas nas categorias menos frequentes.
* **Quantificação de Incerteza (IC):** Utilização de Bootstrapping não-paramétrico com N amostragens para estimar intervalos de confiança a 95% para as métricas de avaliação.
    * Justificativa: Como a distribuição exata das populações de erro dos classificadores é desconhecida, o reamostreamento empírico por Bootstrap é a técnica estatística ideal para inferir a variabilidade e garantir rigor nas métricas reportadas.
* **Teste de Hipótese (Modelo A vs B):** Como as predições são geradas sobre a mesma amostra de chamados (dados pareados/dependentes) e a variável de resposta é categórica binária (acerto ou erro), o Teste de McNemar será aplicado.
    * Justificativa: Testes tradicionais como t-test independente ou Qui-Quadrado assumem independência entre os grupos.

## 3. Roadmap de Execução

### Fase 0: Setup e Reprodutibilidade
- [x] Definir `requirements.txt` com versões exatas.
- [x] Configurar `.gitignore`.
- [x] Criar estrutura de pastas conforme README oficial.

### Fase 1: Análise Exploratória - `01_analise_exploratoria.ipynb`
- [x] Carregar dados sintéticos e validar tipos de colunas.
- [x] Analisar distribuição da `categoria_real` (verificar nível de desbalanceamento).
- [x] Mapear padrões espaciais (`bairro`) e temporais (`data_abertura`).
- [x] Explorar `canal` versus tamanho/complexidade do `texto`.
- [x] Síntese: Registrar os 3-5 achados essenciais que impactam a classificação.

### Fase 2: Auditoria Modelo A - `02_auditoria_modelo_a.ipynb` 
- [x] Calcular Acurácia, Precision, Recall e F1-Score globais e por classe.
- [x] Implementar Bootstrapping para gerar os Intervalos de Confiança das métricas.
- [x] Plotar e interpretar a Matriz de Confusão.
- [x] Analisar calibração: Relação entre `conf_modelo_a` e taxa empírica de acerto.
- [x] Identificar subgrupos de falha (ex: bairros específicos ou canais com maior erro).
- [x] Síntese: Registrar os principais modos de falha e impacto operacional.

### Fase 3: Comparação e Recomendação - `03_comparacao_e_recomendacao.ipynb`
- [ ] Calcular métricas de desempenho para o Modelo B.
- [ ] Executar Teste de McNemar entre Modelo A e Modelo B; calcular e interpretar o p-valor.
- [ ] Avaliar trade-offs de troca: O Modelo B ganha no global, mas perde em categorias críticas?
- [ ] Síntese: Redigir o parágrafo de recomendação executiva evidenciando riscos e limitações.

### Fase 4: Entrega Final
- [ ] Compilar o sumário executivo no `README.md`.
- [ ] Garantir que todos os notebooks rodem de ponta a ponta sem erros.

## 4. Log de Decisões

### Conclusões da Fase 1 (Análise Exploratória)

* **1. Desbalanceamento do Target:**
    * A classe majoritária (`iluminacao_publica`) representa quase 23% dos chamados, enquanto classes minoritárias (como `sinalizacao`) não chegam a 5%.
    * *Impacto no Classificador:* Invalida o uso da Acurácia Global como métrica principal de avaliação na Fase 2. Modelos ingenuos que apenas predigam a classe majoritária apresentarão métricas ilusoriamente altas. Exigirá o uso do F1-Score Macro/Weighted e uma avaliação rigorosa isolada por categoria.
* **2. Independência Estatística dos Metadados (Ruído):**
    * Testes Qui-Quadrado de Pearson comprovaram que a distribuição das categorias de chamados independe da geografia (`bairro`, p-valor ≈ 0.98), do tempo (`dia_semana`, p-valor ≈ 0.46) ou da origem (`canal`, p-valor ≈ 0.84).
    * *Impacto no Classificador:* Define a arquitetura do modelo. O classificador em produção não deve utilizar atributos tabulares como features estruturadas, pois eles agem como ruído e não carregam sinal preditivo real. O peso do sucesso da decisão classificatória recai puramente na extração semântica da variável textual.
* **3. Homogeneidade na Complexidade da Entrada (Texto):**
    * O comprimento das manifestações escritas pelo cidadão (mediana estável em 28 palavras) e a presença de indicadores booleanos de textos longos (`texto_longo`, p-valor ≈ 0.77) permanecem constantes em todos os canais de entrada (App, Telefone, Web).
    * *Impacto no Classificador:* Simplifica o pipeline de engenharia de dados e pré-processamento. Não há necessidade de construir rotinas de Processamento de Linguagem Natural (NLP) separadas ou regras de truncamento específicas dependendo da origem do chamado, já que o modelo enfrentará uma complexidade textual padronizada em toda a rede de captação do 1746.
* **4. Desvios Sistemáticos nas Predições Marginais (Mapeamento de Viés):**
    * A comparação das distribuições marginais revela que os modelos não reproduzem a proporção real das categorias. Há indícios visuais de que um ou ambos os modelos tendem a inflar artificialmente a frequência de predição de certas classes.
    * *Impacto no Classificador:* Esse diagnóstico antecipa distorções que serão investigadas na Fase 2 e Fase 3 através da matriz de confusão.

### Conclusões da Fase 2 (Auditoria do Modelo A)

* **1. Incerteza e Estabilidade Preditiva (Bootstrapping):**
    * O desempenho global (F1-Score Macro de ~0.77) é estatisticamente estável. Contudo, a quantificação de incerteza por categoria comprovou a alta variância em classes com menor suporte amostral (ex: `sinalizacao`), demonstrando que a confiabilidade do modelo cai severamente na ausência de volume de dados.
* **2. Modos de Falha Críticos (Sobreposição Semântica):**
    * A matriz de confusão revelou que o pior desempenho do modelo ocorre na classe `esgoto_vazamento` (Recall ~0.57), cujos falsos negativos são massivamente classificados como `buraco_via`. O modelo falha na desambiguação sintática entre problemas de saneamento e pavimentação.
    * *Impacto Prático:* Gera ineficiência logística por deslocamento de equipes incorretas (Secretaria de Conservação em vez de concessionária de água/esgoto).
* **3. Calibração de Probabilidades (Superconfiança):**
    * O cálculo do Expected Calibration Error (ECE) e o Diagrama de Confiabilidade indicam que o modelo é superconfiante. Quando o sistema reporta >90% de certeza, a acurácia empírica orbita próximo de 80%. As distribuições de probabilidade para predições erradas e corretas possuem alta sobreposição.
    * *Impacto Prático:* O uso da métrica `conf_modelo_a` como gatilho para automação total de despachos (*Straight-Through Processing*) é de alto risco.
* **4. Viés Algorítmico e Subgrupos:**
    * Testes Qui-Quadrado aplicados aos erros comprovaram desempenho homogêneo para bairro (p=0.86) e canal (p=0.36), atestando equidade na prestação do serviço. No entanto, rejeitou-se a hipótese de homogeneidade para o comprimento do texto ($p < 0.05$). Textos do primeiro quartil (Q1, manifestações muito curtas) concentram taxas de erro expressivamente maiores devido à esparsidade de features (falta de contexto semântico).