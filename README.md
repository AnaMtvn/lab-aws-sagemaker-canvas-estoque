# 📊 Previsão de Estoque Inteligente na AWS com [SageMaker Canvas](https://aws.amazon.com/pt/sagemaker/canvas/)



## COMEÇANDO:

Iniciando pelo planejamento, tentei seguir uma linha parecida com o conteúdo da aula prática. Escolhi prever a demanda assim como na aula, me atentando a detalhes que poderiam influenciar na análise dos dados, gerando previsões falsas.

Durante a aula foi possível perceber que as predições foram afetadas a partir do momento em que o valor da quantidade de estoque de um produto era zerada, o modelo passava a considerar a demanda zerada também.

Pensando nisso, nesse ponto me atentei a começar pensando em como poderia contornar o problema da ruptura de estoque e em quais informações poderiam representar algum peso para o treinamento do modelo. Para me ajudar com esse planejamento, consultei o ChatGPT para tirar dúvidas e analisar as possibilidades.

---

## Tratamento de Ruptura de Estoque:

Para tentar solucionar o erro de ruptura de estoque, pensei em um cenário de loja online, onde após o estoque de um produto ser zerado, é habilitado um botão para notificar aos clientes que ainda demonstrarem interesse em comprar o item X, quando este voltar ao estoque.

- Porém, é necessário ter em mente que nem todos os clientes que demonstram interesse realizam a compra de fato. Por esse motivo, o **fator de conversão estimado** representa uma proporção de usuários que, após demonstrar interesse durante períodos de indisponibilidade, efetivamente realizam a compra quando o estoque é reabastecido. Esse fator atua como a ponte entre interesse e compra.

Nesse cenário, espera-se como resultado que:

1. Ruptura de estoque ≠ demanda zero  
2. O interesse/procura pelo produto, transformado em demanda estimada, se torna informação com valor para o treinamento do modelo  
3. Obtenha-se uma melhoria na precisão da previsão de estoque  

---

## SELEÇÃO DO DATASET:

Após pesquisar sobre o assunto com o ChatGPT, escolhi gerar um novo dataset baseado no resultado da conversa.

Para gerar o dataset, utilizei o seguinte prompt:

> Agora, atue como um cientista de dados e crie um dataset em formato CSV com 1000 registros. Esse arquivo deve refletir o histórico de vendas de 25 produtos diferentes contendo colunas de informação sem dados nulos, crie colunas para: ID_PRODUTO (numérico incremental), DIA (a partir de 01/01/2024), MES (para calcular sazonalidade mensal), PREÇO_BASE, DESCONTO_PROMOCAO (em dias de promoção o preço é 10% menor que o preço original), QUANTIDADE_ESTOQUE (nenhum produto deve começar com estoque zerado), DEMANDA_DIA, DEMANDA_LATENTE (para representar os cliques no botão de "avise-me" quando o estoque zerar mas, ainda houver interesse, assumindo que foi registrado um único clique para cada cliente) e DEMANDA_AJUSTADA (utilizando fator de conversão estimado de 0,5).

Nesse contexto:

- A coluna **MES** teria o intuito de agrupar as demandas para possibilitar analisar a variação da demanda durante os períodos.  
  Exemplos:  
  *"Em dezembro a demanda sobe 30%, mesmo sem desconto"* ou  
  *"Produto X tende a ter maior demanda entre os meses 10 e 12, especialmente quando há desconto."*

- Substituí a coluna **FLAG_PROMOCAO** por **DESCONTO_PROMOCAO**, assim como seu valor, que alterei de boolean para um cálculo utilizando uma porcentagem (1 - 0.1).

- Na tentativa de encontrar uma solução para o problema da ruptura de estoque (quando o estoque está zerado), adicionei as colunas **DEMANDA_DIA**, **DEMANDA_LATENTE** e **DEMANDA_AJUSTADA**.

- A **DEMANDA_DIA** representa as vendas do dia, enquanto que a **DEMANDA_LATENTE** atua como indicador de vendas perdidas por falta de estoque, mas que ainda há clientes interessados em adquiri-lo.

- A coluna **DEMANDA_AJUSTADA** utiliza um cálculo para estimar a demanda mesmo quando o estoque é zero, levando em consideração a quantidade de clientes interessados, ou seja, a **DEMANDA_LATENTE**. O resultado é obtido da soma:  
  `DEMANDA_DIA + (DEMANDA_LATENTE * fator de conversão estimado)`.

---

## PROCESSO DE CRIAÇÃO:

Após selecionar o dataset, foram necessárias algumas alterações:

- Mudar o tipo de dados da coluna de ID para `text`, requisito da própria ferramenta.
- Mudar o tipo de dados da coluna **MES** também para `text`, para que esse pudesse ser usado para agrupar os resultados.
- Acabei desconsiderando a coluna **MES** em algumas tentativas, principalmente após considerar que 1000 linhas representam apenas 40 dias, sendo assim, temos apenas 9 dias referentes ao mês 2, o que considero insuficiente para uma análise de demanda sazonal.
- Não foi necessário tratar dados nulos ou inválidos.

Realizei 4 tentativas de treinamento, para todas elas utilizei o tipo de construção **Quick build**:

1. No primeiro utilizei como variável target a coluna **DEMANDA_LATENTE** (pequeno equívoco), desconsiderando a coluna **MES**, com predição para um período de 9 dias. 
![Construção modelo 1 v.1] (<Captura de tela 2026-01-31 184040.png>)
![resultado metricas modelo 1 v.1](<Captura de tela 2026-01-31 190114.png>)

2. Na segunda tentativa, alterei o target para **DEMANDA_AJUSTADA**, que também utilizei nas tentativas subsequentes, considerando agrupamento pela coluna **MES**, com predição para 7 dias.  
![resultado metricas modelo 2](<Captura de tela 2026-01-31 191410.png>)

3. Nesta tentativa, gerei uma segunda versão da segunda tentativa, desconsiderando o agrupamento por **MES**, para 5 dias.  
![resultado metricas modelo 2 v.2] (<Captura de tela 2026-01-31 194500.png>)

4. Na última tentativa, reduzi o número de linhas do dataset utilizadas para treinar o modelo, além de mais uma vez diminuir o período de predição, desta vez para 2 dias.   
![resultado metricas modelo 3] (<Captura de tela 2026-01-31 202945.png>)

---

## RESULTADOS:

Apesar de todos os modelos terem sido gerados, não consegui realizar nenhuma predição, nem mesmo uma *single*, devido a problemas com falta de recursos.

![erro] (<Captura de tela 2026-01-31 192048.png>) 
![erro] (<Captura de tela 2026-01-31 195740.png>)

*Sobre as métricas:* 
- Apesar de não ter certeza se os valores que não foram apresentados nas métricas representam *'0'*, caso isso seja confirmado, os resultados indicam um bom desempenho do modelo.
- Por outro lado, alguns valores estão bastante elevados nos mesmos modelos, o que sugere a necessidade de um melhor balanceamento mas, ainda aparentam ser capazes de apresentar resultados coerentes.
- Os últimos modelos testados aparentam estar mais balanceados em relação a essas métricas.

Por fim, fica pendente, neste momento, a geração dos resultados das predições e uma análise mais aprofundada dessas métricas como próximos passos.

---

## CONSIDERAÇÕES FINAIS:

Gostei muito da experiência apesar de, na prática, não ter conseguido concluí-la de fato devido a não ter conseguido gerar as predições.

É uma área muito interessante e que está se convertendo em uma ferramenta indispensável para todas as áreas de negócios.

A limitação de recursos evidenciou a importância do planejamento de custos e da gestão de quotas ao trabalhar com serviços em nuvem.

A ferramenta da AWS com certeza é incrível, mas acredito que, mesmo depois de seguir o tutorial para evitar as cobranças após realizar os experimentos, precisarei de sorte para não receber uma surpresa na fatura. 🙂

