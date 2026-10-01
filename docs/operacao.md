# Execução e Operação

## Rotina prevista

1. O Cloud Scheduler chama o serviço de ingestão no Cloud Run.
2. O serviço extrai os dados da API, gera um arquivo Parquet e o envia para o Cloud Storage.
3. A tabela externa Raw passa a disponibilizar o arquivo mais recente para consulta no BigQuery.
4. A rotina da Silver identifica inclusões, alterações e remoções, atualizando a tabela com `MERGE`.
5. As dimensões e a tabela fato da Gold são atualizadas.
6. O Power BI atualiza o modelo semântico e os visuais.

## Validação após a carga

Após cada execução, recomenda-se verificar:

- os logs do Cloud Run para confirmar a conclusão da extração;
- a presença do novo arquivo Parquet no bucket;
- o registro mais recente em `Silver.payments_load_audit`;
- a reconciliação entre o total de registros ativos na Silver e a fato na Gold.

## Reprocessamento

Em caso de falha na ingestão, a execução pode ser disparada manualmente no Cloud Run. Após a criação do arquivo no bucket, as consultas de atualização Silver e Gold devem ser executadas novamente na ordem definida.

## Segurança operacional

Tokens e credenciais não devem ser incluídos no repositório. O token da API deve ser fornecido ao Cloud Run por variável de ambiente ou Secret Manager. O Scheduler deve autenticar a chamada ao serviço por IAM/OIDC.

