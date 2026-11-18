# Relatório de Machine Learning: AlertaChuva

**Global Solution 2026.2 · CitizenTech · Inteligência Artificial & Machine Learning**
Integrantes: Danilo Affonso Luz Rios (RM 554791) · Lucca Natario do Vale (RM 95688) · Thiago Feltrin Geraldes (RM 555805)
Cidadão: morador de área de risco de alagamento no **Jardim Pantanal (São Miguel Paulista, zona leste de São Paulo)**.

## 1. Problema e pergunta de ML

Hoje o morador descobre a chuva forte quando ela já está caindo, sem tempo para tirar documentos, móveis e pessoas vulneráveis de casa. O AlertaChuva quer avisá-lo **na véspera**.

**Pergunta:** *com os dados meteorológicos observados hoje pelo INMET, qual a chance de amanhã cair 20 mm ou mais de chuva em São Paulo?*

- **Tipo:** classificação binária (1 = dia seguinte com ≥ 20 mm).
- **Por que 20 mm:** é o índice **R20mm**, "dia de chuva muito intensa", padrão da OMM/ETCCDI para extremos de precipitação. Também medimos o desempenho nos dias **≥ 50 mm**, que correspondem ao aviso **laranja (perigo)** do INMET para chuvas intensas.
- **Indicador do projeto:** antecedência do alerta (≈ 12–24 h, emitido no fim do dia) e quantos eventos foram avisados.

## 2. Dados reais

| | |
|---|---|
| Fonte | INMET, BDMEP: estação automática **A701 – São Paulo/Mirante de Santana** |
| Link | https://bdmep.inmet.gov.br/ |
| Período | **24/07/2006 a 25/04/2025**, diário (6.851 dias) |
| Variáveis | chuva (mm), temperatura média, umidade relativa média, vento médio |

**Integridade:** o arquivo foi conferido contra dois outros exports do mesmo BDMEP: 99,94% dos 1.743 dias e 100% dos 169 meses são idênticos. Detalhes, SHA-256 e passo a passo para baixar novamente estão em `dados/README.md`.

**Achado exploratório:** há em média ≈ 27 dias/ano com ≥ 20 mm, concentrados de dezembro a março (`figuras/01_sazonalidade.png`). A classe é rara: **7,6%** dos dias.

## 3. Preparação e divisão treino/teste

- **Limpeza:** `null` → faltante (≈ 1% por variável); checagens físicas (chuva ≥ 0, umidade de 0 a 100%, temperatura plausível). Dias com dado faltante foram **removidos, sem imputação**, para não fabricar valores: **6.359 dias úteis**.
- **16 variáveis**, todas conhecidas no fim do dia D (sem vazamento do futuro): chuva de hoje e acumulados de 3/7/15/30 dias (solo encharcado), dias consecutivos sem chuva, temperatura e sua anomalia de 30 dias (calor acima do normal favorece temporais), variações de 1 dia de temperatura, umidade e vento (passagem de frente fria), umidade média de 3 dias e sazonalidade cíclica (seno/cosseno do dia do ano).
- **Divisão cronológica**, nunca aleatória: embaralhar dias de uma série temporal deixaria o modelo "ver o futuro".

| Conjunto | Período | Dias | Eventos ≥ 20 mm | Prevalência |
|---|---|---|---|---|
| Treino | ago/2006 – dez/2021 | 5.333 | 401 | 7,5% |
| Teste | jan/2022 – abr/2025 | 1.026 | 81 | 7,9% |

## 4. Escolha do modelo: XGBoost

- Dados **tabulares** com interações não lineares (por exemplo, calor anômalo + umidade subindo + verão). Árvores com *boosting* capturam isso sem normalização.
- Trata **classe rara** com `scale_pos_weight` (≈ 12,3 = negativos/positivos do treino).
- `learning_rate` e profundidade dão controle direto sobre o overfitting.
- Alternativas descartadas: **LSTM** precisaria de bem mais dados que ~6 mil dias; **KMeans** não é supervisionado e não responde à pergunta; **Random Forest** seria viável, mas tende a render menos em eventos raros.

## 5. Treino com diferentes hiperparâmetros

**3 hiperparâmetros × 36 combinações** (o mínimo pedido é 10): `max_depth` ∈ {1, 2, 3, 5}, `learning_rate` ∈ {0,01; 0,03; 0,1} e `n_estimators` ∈ {100, 300, 600}. Fixos: `subsample` e `colsample_bytree` = 0,8 e `min_child_weight` = 5.

**Validação cruzada apenas no treino**, com `TimeSeriesSplit` (5 dobras, cada validação sempre posterior ao seu treino). Métrica de seleção: **Average Precision (PR-AUC)**, adequada à classe rara. Acurácia engana: "nunca vai chover forte" acertaria 92%.

| Rank | depth | lr | n | AP treino | AP validação | Gap |
|---|---|---|---|---|---|---|
| **1** | **3** | **0,01** | **300** | **0,507** | **0,322** | **0,19** |
| 2 | 2 | 0,01 | 300 | 0,410 | 0,321 | 0,09 |
| 3 | 2 | 0,03 | 100 | 0,399 | 0,319 | 0,08 |
| 35 | 5 | 0,1 | 600 | 1,000 | 0,257 | 0,74 |
| 36 | 1 | 0,01 | 100 | 0,249 | 0,229 | 0,02 |

Tabela completa em `resultados/grid_search.csv`. O gráfico `figuras/02_grid_treino_vs_validacao.png` mostra os dois extremos: modelos rasos e com poucos passos ficam **sub-ajustados** (rank 36, treino e validação baixos), e modelos profundos com muitas árvores **decoram o treino** (AP de treino = 1,0, validação cai para 0,26). O escolhido fica no meio-termo.

**Limiar de alerta:** escolhido nas previsões fora da dobra do treino, maximizando o **F2** (recall vale 2× a precisão, porque deixar de avisar custa mais que um alarme falso). Resultado: nível **ALTO ≥ 0,49** e **ATENÇÃO ≥ 0,24**.

## 6. Resultados: treino × teste

| Métrica | Treino | Teste | Diferença |
|---|---|---|---|
| Average Precision (PR-AUC) | 0,406 | **0,270** | 0,14 |
| ROC-AUC | 0,891 | **0,820** | 0,07 |
| Recall (eventos avisados) | 0,873 | **0,778** | 0,10 |
| Precisão | 0,233 | 0,196 | 0,04 |
| F2 | 0,564 | 0,488 | 0,08 |

**Comparação no teste com referências simples:**

| Abordagem | AP | ROC-AUC | Recall |
|---|---|---|---|
| **XGBoost (AlertaChuva)** | **0,270** | **0,820** | **77,8%** |
| Climatologia do mês | 0,133 | 0,676 | n/a |
| Persistência ("choveu ≥ 20 mm hoje") | 0,105 | 0,570 | 21,0% |
| Sorteio | 0,079 | 0,500 | n/a |

**Overfitting:** existe, mas é moderado. A queda treino → teste é de 0,07 em ROC-AUC e 0,10 em recall, e o recall no teste ficou alto (77,8%). O gap em AP (0,14) é esperado: a chuva convectiva de verão é pouco previsível com dados diários de uma estação. Os gráficos estão em `figuras/03_treino_vs_teste.png` (barras, curva PR e matriz de confusão). As variáveis mais importantes foram **variação da umidade em 1 dia**, **sazonalidade** e **dias sem chuva** (`figuras/04_importancia.png`), coerente com a física: umidade subindo antes de frentes e temporais.

## 7. Conclusão: o que isso significa para o cidadão

Resultado no **teste (2022–2025, dados que o modelo nunca viu)**, por nível mostrado no app:

| Nível no app | Dias | Dias que tiveram ≥ 20 mm | % de todos os eventos |
|---|---|---|---|
| 🟢 BAIXO | 373 | 3 (0,8%) | 3,7% |
| 🟡 ATENÇÃO | 331 | 15 (4,5%) | 18,5% |
| 🔴 ALTO | 322 | 63 (19,6%) | **77,8%** |

- **Quando o app diz BAIXO, o morador pode confiar:** só 0,8% desses dias tiveram chuva muito intensa.
- **8 em cada 10 dias de chuva muito intensa foram avisados na véspera** com nível ALTO, e 96% com ATENÇÃO ou ALTO.
- **Dias extremos (≥ 50 mm):** **14 de 16** tiveram aviso na véspera, incluindo o recorde da série, **125,6 mm em 25/01/2025** (índice 0,84 → ALTO).
- **O custo:** cerca de 4 de cada 5 dias em ALTO não chegam a 20 mm. Para quem mora em área de alagamento, o aviso significa "suba os documentos e acompanhe o sensor", ação barata. Por isso o projeto prioriza recall. A frequência real de cada nível é o que o app deve comunicar.

**Limitações e próximos passos:** uma única estação, a ≈ 20 km do bairro, com dados diários, não enxerga temporais isolados. Os 2 extremos perdidos (14/02/2024 e 14/04/2025) foram pancadas sem sinal na véspera. O índice de 0 a 1 não é uma probabilidade calibrada (efeito do `scale_pos_weight`); por isso o app usa níveis e frequências observadas. Evoluções: dados horários e de pressão, estações do CGE/Cemaden na zona leste, e o sensor de nível de água (ESP32) do próprio bairro, que fecha a lacuna de última hora.

*Reprodução:* `pip install -r requirements.txt` → abrir `alertachuva_ml.ipynb` → *Run All* (semente 42, resultados determinísticos).
