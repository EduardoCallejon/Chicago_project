# Chicago Payments Portal

Neste projeto eu construí um pipeline de dados em nuvem a partir da base pública de **Payments** da cidade de Chicago. A ideia foi sair de uma extração simples de API e chegar a uma estrutura que eu consigo explicar, testar e evoluir: ingestão, armazenamento bruto, tratamento incremental, camada analítica e dashboard.

Eu decidi trabalhar com essa base porque ela representa bem um cenário comum de engenharia de dados: a fonte é externa, o volume é suficiente para exigir paginação, os registros podem mudar ao longo do tempo e os dados precisam ser organizados para análise financeira.

## Objetivo

O objetivo do projeto é disponibilizar uma visão confiável dos pagamentos públicos de Chicago, permitindo análises como:

- valor total pago ao longo do tempo;
- quantidade de pagamentos;
- fornecedores com maior valor recebido;
- departamentos com maior volume financeiro;
- contratos e pagamentos por período.

## Arquitetura atual

```text
Chicago Data Portal API
        |
        v
Cloud Run Function
        |
        v
Google Cloud Storage - Raw Parquet
        |
        v
BigQuery Raw external table
        |
        v
BigQuery Silver incremental MERGE
        |
        v
BigQuery Gold dimensional model
        |
        v
Power BI dashboard
```

O Cloud Scheduler inicia a ingestão. Depois, as consultas agendadas atualizam Silver e Gold. Neste momento a sequência é controlada por horários; como evolução, o fluxo pode ser orquestrado pelo Cloud Workflows, aguardando o sucesso de cada etapa antes de iniciar a próxima.

## Fase 1 - Extração da API

Eu utilizo a API Socrata do Chicago Data Portal:

```text
https://data.cityofchicago.org/api/v3/views/s4vu-giwb/query.json
```

O arquivo `src/Extract.py` faz a extração com `requests` e `polars`. A API é consumida em páginas de 5.000 linhas para evitar uma única requisição grande demais. A cada página concluída, o processo escreve no log a quantidade do lote e o acumulado processado.

Também criei a coluna técnica `source_row_id` a partir do identificador retornado pela fonte. Ela é a chave usada para localizar cada pagamento nas cargas seguintes.

```sql
SELECT
  *,
  :id AS source_row_id
ORDER BY :id
```

## Fase 2 - Camada Raw

Depois da extração, os dados são gravados em Parquet com compressão Zstandard. Localmente, o arquivo fica em `data/raw/payments.parquet`. No Cloud Run, ele é criado em `/tmp/payments.parquet`, pois o armazenamento do container é temporário.

Em seguida, o arquivo é enviado para o Google Cloud Storage:

```text
gs://<GCS_BUCKET_NAME>/raw/payment_YYYY-MM-DD.parquet
```

O Parquet é a minha camada Raw: ele mantém o formato recebido da fonte e permite reprocessamentos futuros. A tabela externa `Raw.payments_external` no BigQuery lê os arquivos diretamente do bucket, sem copiar o conteúdo para outra tabela física.

## Fase 3 - Silver incremental

Na Silver eu limpo e tipifico os campos mais importantes:

- removo espaços e valores vazios de campos textuais;
- converto `amount` para `NUMERIC`;
- converto `check_date` para `DATE`;
- mantenho `source_row_id` como chave técnica.

Para identificar mudanças de conteúdo, eu gero um `record_hash` com os campos relevantes do pagamento. O ID responde qual registro é aquele; o hash responde se os dados daquele registro mudaram.

O `MERGE` executa três ações:

1. faz `INSERT` quando o pagamento ainda não existe na Silver;
2. faz `UPDATE` quando o ID existe, mas o hash mudou;
3. faz `DELETE` quando o pagamento existia na Silver, mas não veio na fotografia atual da API.

Essa última regra só faz sentido porque a extração traz a fotografia completa da fonte. Para reduzir risco, adicionei validações antes do `MERGE`: a carga não pode ter IDs nulos ou duplicados e não pode cair abruptamente abaixo de 95 por cento do volume da Silver anterior.

Também mantenho a tabela `Silver.payments_load_audit`, que registra o arquivo processado e as quantidades de linhas inseridas, atualizadas, mantidas e excluídas. Isso facilita entender o que ocorreu em cada execução sem depender apenas de logs.

## Fase 4 - Gold para análise

Depois que a Silver ficou consistente, criei uma Gold no formato estrela. Para o volume atual, decidi recriar essa camada diariamente com `CREATE OR REPLACE TABLE`. A Silver continua incremental; a Gold é derivada da Silver e pode ser reconstruída de maneira simples e segura.

```text
Gold
├── dim_vendor
├── dim_department
├── dim_calendar
└── fact_payments
```

### Dimensões

- `dim_vendor`: uma linha por fornecedor;
- `dim_department`: uma linha por departamento;
- `dim_calendar`: uma linha por data, com atributos de ano, trimestre, mês e dia.

Como a fonte não oferece IDs próprios para fornecedor e departamento, eu gero `vendor_key` e `department_key` a partir dos nomes normalizados com `UPPER` e `TRIM`.

Durante a construção, encontrei uma duplicidade em `dim_vendor`: nomes com variações de caixa ou espaços produziam a mesma chave, mas permaneciam como linhas diferentes. Isso multiplicava pagamentos no `JOIN` da fato. Corrigi a dimensão normalizando o nome antes do `DISTINCT` e validei novamente a fato.

### Tabela fato

`Gold.fact_payments` possui uma linha por pagamento. Ela contém a chave do pagamento, data, valor, contrato e as chaves de fornecedor e departamento.

Uma validação importante foi comparar os volumes:

```text
Silver ativa: 467.980 pagamentos
Gold fact_payments: 467.980 pagamentos
IDs distintos na fato: 467.980
```

Com isso, confirmei que a fato não possui duplicidades e que o valor total pago não está sendo inflado por joins.

## Dashboard

O projeto possui o arquivo `Dash_board_chicago.pbix` como ponto de partida para o dashboard. Também criei um layout editável no Figma com os blocos principais:

- Total Pago;
- Pagamentos;
- Fornecedores;
- Departamentos;
- Distribuição de Pagamentos por Período;
- Top Fornecedores por Valor Pago;
- Pagamentos por Departamento;
- Distribuição por Faixa de Valor.

O arquivo Figma tem páginas para dashboard, tokens visuais e componentes reutilizáveis. O acesso de edição depende de a conta ter um assento Full ou Design; uma conta com assento View consegue abrir o arquivo, mas não consegue alterar os elementos.

## Execução local

### Pré-requisitos

- Python 3.14;
- `uv`;
- token gratuito da API Socrata;
- uma conta Google Cloud apenas se o upload para o bucket for usado.

Crie um arquivo `.env` na raiz do projeto:

```text
SOCRATA_APP_TOKEN=seu_app_token
GCS_BUCKET_NAME=nome-do-seu-bucket
```

Instale as dependências e execute:

```powershell
uv sync
uv run python main.py
```

Sem `GCS_BUCKET_NAME`, a extração ainda roda e o Parquet permanece localmente em `data/raw/payments.parquet`.

## Cloud Run

O `main.py` possui dois modos de uso:

- `main()`: execução local para desenvolvimento e testes;
- `ingest_payments(request)`: endpoint HTTP chamado pelo Cloud Scheduler.

O `Dockerfile` usa `functions-framework` e escuta a porta definida pela variável `PORT` do Cloud Run. Em produção, o token Socrata deve vir do Secret Manager como `SOCRATA_APP_TOKEN`; ele não deve ser salvo no repositório.

A Service Account do Cloud Run precisa de permissão para gravar objetos no bucket. O Scheduler deve chamar o endpoint usando OIDC/IAM, em vez de deixar o serviço público.

## Próximas evoluções

Os próximos passos que pretendo desenvolver são:

1. substituir a dependência de horários fixos por Cloud Workflows;
2. organizar as transformações SQL em modelos dbt quando a Gold crescer;
3. incluir testes de qualidade automáticos para chaves, volume, nulos e reconciliação;
4. adicionar alertas de falha e monitoramento de custos;
5. evoluir o dashboard para uma interface conversacional que responda perguntas sobre os dados.

## Estrutura do repositório

```text
Chicago_portal
├── main.py                    # Ponto de entrada local e HTTP
├── Dockerfile                 # Imagem para Cloud Run
├── src
│   ├── Extract.py             # Paginação e extração com Polars
│   └── load.py                # Parquet local e upload para GCS
├── data
│   └── raw                    # Saída local de desenvolvimento
├── Dash_board_chicago.pbix    # Dashboard Power BI
└── Guia_Arquitetura_Cloud_Run_Chicago.docx
```
