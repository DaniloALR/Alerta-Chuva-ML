# Dados: AlertaChuva (ML)

Todos os dados são **reais e públicos**, do **INMET (Instituto Nacional de Meteorologia)**. Nenhum dado foi simulado.

## Fonte principal (usada no treino e no teste)

| Campo | Valor |
|---|---|
| Órgão | INMET, Banco de Dados Meteorológicos (BDMEP) |
| Link oficial | https://bdmep.inmet.gov.br/ (exportação "Dados Diários", estação automática) |
| Estação | **A701 – São Paulo / Mirante de Santana** (lat -23,496; lon -46,620; alt. 785,6 m) |
| Periodicidade | Diária |
| Período | **24/07/2006 a 25/04/2025** (6.851 dias) |
| Variáveis | Precipitação total diária (mm), temperatura média (°C), umidade relativa média (%), velocidade média do vento (m/s) |
| Arquivo | `raw/dados_A701_D_2006-07-24_2025-04-25.csv` |
| SHA-256 | `0663a8bc4ef0e9a7d0e80b12728ff23874e0cd222db9a64f15fdf095d7915463` |

**Por que a A701:** é a estação de referência do INMET para a cidade de São Paulo, com a série automática mais longa. O Jardim Pantanal não tem estação própria, e a chuva medida no Mirante serve como sinal regional. A leitura local fica a cargo do sensor ESP32 do projeto.

## Arquivos de verificação de integridade

Usados **apenas para conferir** que o arquivo principal é o dado oficial e não foi alterado:

| Arquivo | Conteúdo | Resultado da conferência |
|---|---|---|
| `raw/dados_A701_D_2020-01-01_2024-11-20.csv` | Outro export diário do BDMEP, mesma estação | 1.743 dias comparados, **99,94% idênticos** |
| `raw/dados_A701_M_2006-07-24_2020-12-31.csv` | Export **mensal** do BDMEP, mesma estação | 169 meses comparados, **100% idênticos** (soma diária = total mensal) |

SHA-256: `02e0a7df…ee6e60` (diário 2020–2024) e `32592b21…b3a076` (mensal).

## Como os arquivos foram obtidos

O portal do INMET exige exportação manual (formulário com e-mail). Os três CSVs estão **no formato original de exportação do BDMEP** (cabeçalho de 10 linhas + `;` + `null`) e foram obtidos de exports públicos já feitos dessa mesma estação:

- Diário 2006–2025: `github.com/wmpjrufg/climatetwin` → `BD/dados/wander/dados_A701_D_2006-07-24_2025-04-25.csv`
- Diário 2020–2024: `github.com/valtency/labmet` → `data/dados_A701.csv`
- Mensal 2006–2020: `github.com/angeruzzi/Datasource` → `pluviometricos/automaticas/dados_A701_M_2006-07-24_2020-12-31.csv`

A coincidência entre três exports independentes (tabela acima) comprova a autenticidade.

**Para reproduzir direto da fonte:** BDMEP → *Dados Diários* → *Estação Automática* → **A701** → período 24/07/2006 a 25/04/2025 → variáveis de precipitação, temperatura média, umidade média e vento médio → exportar CSV.

## Tratamento aplicado (detalhes no notebook)

1. `null` → valor faltante (≈1% por variável). O calendário foi completado dia a dia.
2. Checagens físicas: chuva ≥ 0, umidade entre 0 e 100%, temperatura entre 0 e 40 °C.
3. Linhas com variável ou alvo faltante foram removidas, sem imputação, para não inventar dados: 6.359 dias úteis.
4. Alvo: `chuva(D+1) ≥ 20 mm` (índice R20mm da OMM/ETCCDI).
