# Arquitetura da Solução

## Visão geral

Este projeto implementa um pipeline de dados para ingestão e análise dos pagamentos disponibilizados pelo Chicago Data Portal. A solução foi organizada em camadas para separar a captura dos dados, o tratamento analítico e o consumo em relatórios.

## Fluxo de dados

```text
Chicago Data Portal API
        |
        v
Cloud Run
        |
        v
Cloud Storage (arquivos Parquet / Raw)
        |
        v
BigQuery Raw (tabela externa)
        |
        v
BigQuery Silver (dados tipados e atualizados)
        |
        v
BigQuery Gold (fato e dimensões)
        |
        v
Power BI
```

## Componentes

### Extração

O Cloud Run executa a rotina Python de ingestão. A API é consultada de forma paginada para reduzir o risco de falhas durante a obtenção de grandes volumes. Ao término da extração, os registros são gravados em formato Parquet.

### Camada Raw

Os arquivos Parquet são armazenados no Cloud Storage. A tabela externa `Raw.payments_external` permite que o BigQuery consulte os arquivos sem a necessidade de copiá-los imediatamente para uma tabela nativa.

### Camada Silver

A tabela `Silver.payments` contém os dados tratados. Nessa etapa são aplicadas conversões de tipo, padronização de campos textuais, cálculo do hash de conteúdo e atualização incremental por meio de `MERGE`.

O campo `source_row_id` identifica o registro de origem. O campo `record_hash` representa o conteúdo relevante do pagamento e permite identificar alterações entre execuções.

### Camada Gold

A camada Gold disponibiliza o modelo analítico utilizado pelo Power BI:

- `Gold.fact_payments`: tabela de fatos com os pagamentos ativos;
- `Gold.dim_vendor`: dimensão de fornecedores;
- `Gold.dim_department`: dimensão de departamentos;
- `Gold.dim_calendar`: dimensão de calendário.

### Orquestração

O Cloud Scheduler inicia a ingestão no Cloud Run. As transformações no BigQuery são executadas após a conclusão esperada da carga. Em um cenário de maior complexidade, essa dependência pode ser substituída por Cloud Workflows ou por uma ferramenta de orquestração dedicada.

