# Dashboard e Métricas

## Objetivo

O dashboard apresenta uma visão consolidada dos pagamentos, permitindo analisar volume financeiro, quantidade de pagamentos, fornecedores, departamentos e evolução temporal.

## Modelo analítico

O Power BI utiliza `Gold.fact_payments` como tabela central. As dimensões de fornecedor, departamento e calendário se relacionam com a fato para permitir filtros e segmentações consistentes.

## Indicadores principais

| Indicador | Definição |
|---|---|
| Total Pago | Soma de `amount` para todos os pagamentos disponíveis no contexto do relatório. |
| Quantidade de Pagamentos | Quantidade de pagamentos distintos na tabela fato. |
| Quantidade de Fornecedores | Quantidade distinta de fornecedores associados aos pagamentos. |
| Quantidade de Departamentos | Quantidade distinta de departamentos associados aos pagamentos. |
| Variação anual | Comparação do total pago com o mesmo período do ano anterior. |
| Variação mensal | Comparação do total pago com o mês imediatamente anterior. |

## Observações de interpretação

O cartão de Total Pago inclui registros sem data de pagamento. Já os gráficos temporais consideram somente registros com `check_date` válida. Essa separação evita que valores sem referência temporal distorçam a leitura mensal.

Para eixos temporais com mais de um ano, recomenda-se utilizar `month_start` ou `year_month`. O campo `month_name` deve ser ordenado por `month_number` para manter a sequência cronológica.

