# Consultas SQL
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-316192?style=for-the-badge&logo=postgresql&logoColor=white)

Considere a exitência de estabelecimento que desejam comprar maquininhas terminais Point on Sale (POS) para executar transações de cartão e assim cobrarem seus clientes. Os estabelecimento encomendam os POS em pedidos, os pedidos têm atualizações e quando os POS chegam ao estabelecimento ocorrem as transações.

* Considere que todos os problemas não precisam ser resolvidos em um único select.
* Cada problema exige sua estrura desejada do resultado da query.

Observações: Há dois modelos de terminal ('d150' e 's920'). Há quatro possíveis status: 'new', 'invoiced', 'transit' e 'delivered'). A variável 'amount_cents' é o valor da venda em centavos.

## Tabelas

Tabela Estabelecimentos
table: merchants

| Name          | Data type         |
| ------------- | ----------------- |
| id            | BigInt            |
| cnpj          | BigInt            |
| razao_social  | Character Varying |
| created_at    | Timestamp without time zone |

Tabela de Pedidos
table: orders

| Name            | Data type         |
| --------------- | ----------------- |
| id              | BigInt            |
| merchant_id     | BigInt            |
| terminal_model  | enum              |
| created_at      | Timestamp without time zone |


* enum: restringe a escolha de dois modelos de terminal, descritos nas observações.


Tabela de Status do Pedido
table: order_status

| Name            | Data type         |
| --------------- | ----------------- |
| id              | BigInt            |
| order_id        | BigInt            |
| status          | enum1             |
| created_at      | Timestamp without time zone |

* enum1: restringe a escolha de quatro status, descritos nas observações

Tabela de Venda
table: transactions

| Name            | Data type         |
| --------------- | ----------------- |
| id              | BigInt            |
| merchant_id     | BigInt            |
| terminal_model  | enum              |
| amount_cents    | BigInt            |
| created_at      | Timestamp without time zone |


## Análises

### Cáculo TPV de vendas por mês. [Colunas: mes | tpv_reais]

```
SELECT extract(month from created_at) AS mes, SUM(amount_cents/100) AS TPV_reais
FROM mkt."TB_transactions" 
GROUP BY EXTRACT(month from created_at)
```
### Lista dos estabelecimentos que nunca fizeram nenhum pedido de terminal. [Colunas: merchant_id | cnpj]
```
SELECT id FROM mkt."TB_merchants" 
EXCEPT
SELECT merchant_id  FROM mkt."TB_orders"
```
### Cálculo da soma do TPV das vendas dos estabelecimentos que possuem mais de dez vendas. [Colunas: merchant_id | tpv_reais]
```
SELECT merchant_id, SUM(amount_cents/100) AS TPV_reais
FROM mkt."TB_transactions" 
GROUP BY merchant_id 
HAVING COUNT(merchant_id) >= 10 
```
### Cálculo TPV por dia por modelo. [Colunas: data | tpv_reais_d150 | tpv_reais_s920]
```
SELECT EXTRACT(data from created_at) AS data, terminal_model, (amount_cents/100) AS TPV_reais
FROM mkt."TB_transactions" 
ORDER BY terminal_model
```

 /* Com o código acima não consegui obter o resultado esperado, por esse motivo testei mais duas consultas diferentes que infelizmente não solucionaram este problema. Aceito sugestões 🤓 */
 
/* Retorna o montante por dia
```
SELECT EXTRACT(data from created_at) AS data,
SUM (
	CASE WHEN terminal_model = 'd150' THEN amount_cents
	ELSE amount_cents end)
FROM mkt."TB_transactions"
GROUO BY data */
```

/* Retorna o montante do amount_cents por terminal_model
```
SELECT 
CASE 
	WHEN terminal_model = 'd150' THEN SUM(amount_cents) 
	ELSE SUM(amount_cents)
	END
FROM mkt."TB_transactions"
GROUP BY terminal_model */
```
### Lista ids e datas dos pedidos cujo o último status é 'transit'. [Colunas: order_id | data]
```
SELECT order_id, created_at AS date
FROM mkt."TB_order_status" 
WHERE status = 'transit'
```
### Lista da quantidade de orders em cada status. [Colunas: status | count]
```
SELECT status, COUNT(status)
FROM mkt."TB_order_status" 
GROUP BY status
```
### Lista de orders cujo o status 'delivered' ocorreu mais de uma semana após o status 'new'. [Colunas: order_id | date_diff_in_days]
```
SELECT order_id, TRUNC(DATE_PART('day', created_at::timestamp - created_at::timestamp)/7) AS semana
FROM mkt."TB_order_status" 
WHERE status = 'delivered'
```
### Calculo do TPV diario de vendas D-30. [Colunas: data | tpv_reais]
```
SELECT EXTRACT(date from created_at) AS data, (amount_cents/100) AS TPV_reais
FROM mkt."TB_transactions" 
WHERE created_at >= NOW() - interval '30 days'
```
