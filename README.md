# APIs, energias renováveis e aprendizado de máquina
- Mauricio Bertuci Saletti - RM571229
- Lucas Caram Bueno - RM570158
- Rhuan Pacheco Carreri - RM570129
- Leonardo Fortini Marcelo - RM572566
- Nicolas Andrade Rodrigues - 572782

## Objetivo

Consultar duas APIs públicas e resolver duas tarefas independentes, comparando três algoritmos em cada uma:

1. **Classificação:** prever a fonte renovável (Solar, Eólica ou Hidráulica) de empreendimentos de geração da ANEEL a partir de potência outorgada, latitude e longitude.
2. **Regressão:** estimar a radiação solar horária (W/m²) em Petrolina (PE) a partir de temperatura, umidade, nuvens, vento e hora do dia.

## Dados

| Tarefa | Fonte | Detalhes |
|---|---|---|
| Classificação | [SIGA — ANEEL](https://dadosabertos.aneel.gov.br/dataset/siga-sistema-de-informacoes-de-geracao-da-aneel) (API CKAN/DataStore) | Até 1.200 registros por sigla (UFV, EOL, UHE, PCH, CGH); 3876 empreendimentos válidos em `aneel_classificacao_orange.csv`. Consulta feita sem token. |
| Regressão | [Open-Meteo Historical Weather API](https://open-meteo.com/en/docs/historical-weather-api) | Petrolina (PE), latitude -9,39 e longitude -40,50, de 01/04/2025 a 30/06/2025, fuso America/Recife, horas entre 7h e 17h; 1001 linhas em `meteo_regressao_orange.csv`. Dados estimados por modelos/reanálise, não medidos em painel fotovoltaico. |

Nenhuma das consultas exige credenciais.

## Como executar

```bash
pip install -r requirements.txt
jupyter notebook avaliacao_apis_energia_ml.ipynb
```

Execute todas as células na ordem (Kernel > Restart & Run All). É necessário acesso à internet para consultar as APIs. Os dois CSVs gerados estão no repositório; ao rodar o notebook eles são regenerados e os números da consulta da ANEEL podem variar levemente se o cadastro for atualizado. Figuras ficam em `figuras/` e os resultados em `resultados_classificacao.csv` e `resultados_regressao.csv`.

## Estrutura

- `avaliacao_apis_energia_ml.ipynb`: consulta às APIs, exploração, modelos, métricas, gráficos e interpretação
- `aneel_classificacao_orange.csv` e `meteo_regressao_orange.csv`: dados gerados pelas APIs
- `resultados_classificacao.csv` e `resultados_regressao.csv`: tabelas comparativas
- `figuras/`: gráficos gerados pelo notebook
- `requirements.txt`: dependências

## Tarefa 1 — Classificação da fonte

**Configuração de avaliação:** divisão 80% treino / 20% teste, estratificada por classe, `random_state=42`. Padronização ajustada somente no treino (Regressão Logística e KNN, com `log1p` na potência). Precision, Recall e F1 com média **macro**; F1 weighted como referência.

Algoritmos: Regressão Logística, KNN (k = 7) e Random Forest.

| Algoritmo | Accuracy | Precision (macro) | Recall (macro) | F1 (macro) | F1 (weighted) |
|---|---|---|---|---|---|
| Regressão Logística | 0.8222 | 0.8367 | 0.8207 | 0.8184 | 0.8219 |
| KNN (k=7) | 0.9665 | 0.9670 | 0.9649 | 0.9658 | 0.9664 |
| Random Forest | 0.9768 | 0.9780 | 0.9755 | 0.9765 | 0.9768 |

**Média das métricas.** Precision, Recall e F1 usam média *macro*; o F1 *weighted* é informado como referência.

**Modelo escolhido.** Random Forest, com o maior F1 macro (0.977) e acurácia de 0.977. Ranking por F1 macro: Random Forest (0.977) > KNN (k=7) (0.966) > Regressão Logística (0.818).

**Classes mais confundidas.** No melhor modelo, o erro mais frequente é Solar previsto como Hidráulica: 7 casos, 2.9% dos exemplos reais de Solar no teste. A classe com menor recall é Solar (95.0%).

## Tarefa 2 — Regressão da radiação solar

**Configuração de avaliação:** primeiras 80% das horas para treino e últimas 20% para teste, com ordem temporal preservada. Entradas: temperatura, umidade, nuvens, vento e hora; `radiacao_w_m2` não entra em X.

Algoritmos: Regressão Linear, Random Forest e Gradient Boosting.

| Algoritmo | MAE (W/m²) | MSE ((W/m²)²) | R² |
|---|---|---|---|
| Regressão Linear | 145.2049 | 30034.2011 | 0.3598 |
| Random Forest | 66.3994 | 7210.0838 | 0.8463 |
| Gradient Boosting | 64.9453 | 7233.5866 | 0.8458 |

**Modelo escolhido.** Gradient Boosting, com o menor MAE no teste (64.9 W/m², cerca de 17.4% da radiação média do período de teste, 373 W/m²) e R² de 0.846. Ranking por MAE: Gradient Boosting (64.9) > Random Forest (66.4) > Regressão Linear (145.2).

**Papel da hora.** Pela importância por permutação, `hora` ocupa a posição 1 de 5 variáveis (aumento de 114.4 W/m² no MAE quando embaralhada); a variável mais importante é `hora`. A hora funciona como proxy da geometria solar: o ângulo do sol define o formato de sino da radiação ao longo do dia, e as demais variáveis modulam esse padrão (principalmente a nebulosidade). O maior erro médio ocorre às 10h (100.3 W/m²).

**Cuidado com a divisão temporal.** A radiação média no treino foi 498 W/m² e no teste 373 W/m²; o teste corresponde ao fim do período, então mudanças sazonais entre abril e junho afetam a comparação.

## Limitações

- **Classificação:** a potência é a capacidade outorgada, não energia gerada; potência e localização não distinguem completamente as fontes, as coordenadas são aproximadas, complexos podem repetir coordenadas e a amostra por sigla é limitada.
- **Regressão:** radiação (W/m²) estimada por modelos não equivale à geração elétrica de um sistema fotovoltaico, que depende de eficiência, área, inclinação, temperatura das células, inversor e perdas. O período cobre apenas três meses e um único local.
