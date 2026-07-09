# Auditoria de Classificadores 1746 - Casa Civil / IplanRio

Este repositório contém a resolução do desafio técnico. O objetivo do projeto foi auditar o classificador de chamados atual da prefeitura (Modelo A) e avaliar, com rigor estatístico, se a sua substituição por uma nova versão (Modelo B) traria ganhos reais para a operação da Central 1746.

## Como Reproduzir

Para garantir que o código rode perfeitamente em qualquer máquina, o projeto foi isolado com dependências exatas. Siga os passos no terminal:

1. Clone este repositório.
2. Crie um ambiente virtual na raiz do projeto: `python -m venv .venv`
3. Ative o ambiente virtual:
* No Windows: `.venv\Scripts\Activate.ps1` (ou `.bat` no CMD)
* No Linux/Mac: `source .venv/bin/activate`


4. Instale as dependências: `pip install -r requirements.txt`
5. Abra o Jupyter (`jupyter notebook`) e execute os três notebooks localizados na pasta `notebooks/` em ordem sequencial (01, 02 e 03).

## Abordagem Metodológica

A análise evitou o uso de métricas simples que costumam mascarar falhas. Como os chamados são muito desbalanceados (muitos sobre iluminação, poucos sobre sinalização), focamos no F1-Score para garantir que problemas raros tivessem peso na avaliação.

Para termos certeza de que os resultados não eram obra do acaso, utilizamos simulações estatísticas (Bootstrapping) para criar intervalos de confiança. Na hora de comparar os dois modelos frente a frente, aplicamos o Teste de McNemar, ideal para cenários onde dois algoritmos avaliam exatamente os mesmos dados. Por fim, investigamos não apenas quem acerta mais, mas quem é mais confiável para operar sozinho usando métricas de calibração de probabilidade.

---

## Sumário Executivo

Nossa investigação começou com o entendimento do comportamento dos cidadãos e das características dos chamados. Descobrimos que atributos como o bairro da ocorrência, o dia da semana ou o canal utilizado (app ou telefone) não ajudam a prever qual é o problema real. O peso da decisão do modelo recai 100% sobre o texto escrito pelo cidadão. Como os textos costumam ter um tamanho parecido em todos os canais, isso simplificou nossa engenharia de dados, permitindo focar diretamente na capacidade de leitura dos modelos.

Ao auditar o modelo atual em produção (Modelo A), descobrimos que sua performance global escondia falhas logísticas graves. O modelo era ruim em ler reclamações muito curtas e sofria de uma confusão estrutural: ele frequentemente classificava vazamentos de esgoto como buracos na via, o que na prática envia equipes de asfalto para resolver problemas de encanamento. Pior ainda, o Modelo A sofria de "superconfiança". Ele errava com o mesmo grau de certeza matemática com que acertava, o que torna muito perigoso deixá-lo despachar equipes automaticamente sem supervisão humana.

O Modelo B, por sua vez, provou ser estatisticamente superior em nossos testes. Ele corrigiu o problema da leitura de textos curtos e també, eliminou a confusão entre esgoto e buracos na via. Além de acertar mais, o Modelo B possui melhores resultados quando citado sua própria incerteza. Conseguimos provar que quando ele está confuso, sua nota de confiança cai. Isso é um ganho imenso para a operação, pois nos permite criar uma esteira automática segura: podemos deixar o modelo agir sozinho quando tem mais de 80, 90% de certeza, e mandar apenas as dúvidas para um atendente humano analisar, enquanto no modelo A isso não era possível.

**Recomendação Executiva (Veredito)**

**Recomendação:** Devemos substituir o Modelo A pelo Modelo B na operação da Central 1746.

**Justificativa e Ganhos:** A adoção do Modelo B entrega um salto de desempenho global estatisticamente comprovado. Do ponto de vista logístico, o principal ganho estrutural é a eliminação do gargalo entre problemas de saneamento (`esgoto_vazamento`) e pavimentação (`buraco_via`), cessando o roteamento incorreto de chamados de água para a Secretaria de Conservação e evitando o deslocamento inútil de maquinário. Além disso, a melhor calibração do modelo permite a adoção de uma esteira de automação segura: ao estabelecer um limiar de 80% de confiança, o sistema despacha automaticamente a maioria dos chamados com alta precisão e desvia apenas as predições incertas para a triagem humana, otimizando o tempo dos atendentes e reduzindo custos operacionais.

**Riscos e Medidas de Mitigação:** A troca de arquitetura apresenta um único custo operacional mapeado: a degradação preditiva na categoria `poda_arvore`. Os erros desta classe passaram a se pulverizar em outras categorias em vez de acertar o alvo. Como **medida de mitigação**, sugere-se a criação de uma regra sistêmica de segurança durante os primeiros meses de implantação: chamados que contenham um vocabulário associado à arborização (árvore, galho, raiz), mas que o modelo classifique com baixa confiança para outras secretarias, devem ser desviados para a fila de triagem humana antes do despacho definitivo.