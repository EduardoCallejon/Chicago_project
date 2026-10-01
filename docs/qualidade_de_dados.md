# Qualidade de Dados

## Regras aplicadas

1. `source_row_id` não pode ser nulo nem duplicado no conjunto usado pela carga.
2. O valor de `amount` é convertido para tipo numérico de forma segura.
3. Campos textuais passam por remoção de espaços excedentes e conversão de valores vazios para nulo.
4. O campo `check_date` é convertido para data quando o formato recebido é válido.
5. O hash do registro é recalculado a partir dos atributos de negócio para identificar mudanças de conteúdo.
6. A Gold considera apenas registros ativos da Silver.

## Registros sem data

Pagamentos sem `check_date` não devem ser descartados do total financeiro. Entretanto, não podem ser distribuídos corretamente ao longo do tempo. Por esse motivo:

- permanecem incluídos em indicadores financeiros gerais;
- ficam fora dos gráficos por mês, trimestre ou ano;
- devem ser acompanhados como ponto de qualidade da fonte.

## Validações recomendadas

```sql
-- Quantidade de registros ativos na Silver
SELECT COUNT(*)
FROM `project-ee331d6f-13ea-4843-b3b.Silver.payments`
WHERE is_active = TRUE;

-- Registros com data ausente
SELECT COUNT(*) AS records_without_date, SUM(amount) AS amount_without_date
FROM `project-ee331d6f-13ea-4843-b3b.Silver.payments`
WHERE is_active = TRUE
  AND check_date IS NULL;

-- Reconciliação entre Silver e Gold
SELECT COUNT(*) AS total_gold
FROM `project-ee331d6f-13ea-4843-b3b.Gold.fact_payments`;
```

