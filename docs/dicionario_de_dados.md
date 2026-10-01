# Dicionário de Dados

## Silver.payments

| Campo | Descrição |
|---|---|
| `source_row_id` | Identificador técnico do registro na fonte de origem. |
| `voucher_number` | Número do voucher associado ao pagamento. |
| `amount` | Valor financeiro do pagamento. |
| `check_date` | Data de emissão ou processamento do pagamento, quando disponível na origem. |
| `contract_number` | Número do contrato relacionado ao pagamento. |
| `department_name` | Nome do departamento responsável. |
| `vendor_name` | Nome do fornecedor beneficiado. |
| `record_hash` | Hash calculado a partir dos campos de negócio para identificação de alterações. |
| `created_at` | Data e hora de inserção do registro na Silver. |
| `updated_at` | Data e hora da última atualização do registro. |
| `is_active` | Indica se o registro permanece ativo na versão mais recente da carga. |

## Gold.fact_payments

| Campo | Descrição |
|---|---|
| `payment_id` | Identificador do pagamento, derivado de `source_row_id`. |
| `voucher_number` | Número do voucher. |
| `contract_number` | Número do contrato. |
| `check_date` | Data do pagamento utilizada nas análises temporais. |
| `amount` | Valor do pagamento. |
| `vendor_key` | Chave de relacionamento com `Gold.dim_vendor`. |
| `department_key` | Chave de relacionamento com `Gold.dim_department`. |
| `created_at` | Data e hora de criação na Silver. |
| `updated_at` | Data e hora da última atualização na Silver. |

## Dimensões

`Gold.dim_vendor` contém a chave e o nome padronizado de cada fornecedor. `Gold.dim_department` contém a chave e o nome padronizado de cada departamento.

`Gold.dim_calendar` contém um registro para cada data do intervalo disponível, com atributos como ano, trimestre, número e nome do mês, dia da semana e início de cada período.

