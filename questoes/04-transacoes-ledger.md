# Questão 04 — Relatório mensal do Ledger

## Prompt

```text
[Task]
Escreva uma query PostgreSQL para consolidar transações por mês e categoria.
Data de referência fictícia do exercício: 2026-04-24. Não use a data atual
real. Recorte: últimos seis meses corridos, não seis meses-calendário fechados.
Tabelas:
CREATE TABLE customers (
 id BIGSERIAL PRIMARY KEY, segment VARCHAR(16) NOT NULL,
 country CHAR(2) NOT NULL, signup_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE TABLE transactions (
 id BIGSERIAL PRIMARY KEY,
 customer_id BIGINT NOT NULL REFERENCES customers(id),
 category VARCHAR(32) NOT NULL, amount_cents BIGINT NOT NULL,
 status VARCHAR(16) NOT NULL, payment_method VARCHAR(16),
 created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(), completed_at TIMESTAMPTZ
);
Índices existentes: created_at, status e category, individualmente.
Categorias: subscription, one_time, refund e credit_adjustment.

[Action]
Filtre status='completed' e use created_at como eixo temporal. Traga mês
YYYY-MM, categoria, quantidade e soma de amount_cents em reais com duas
casas decimais, sem divisão inteira. Ordene mês e categoria crescentes.
Preserve o sinal armazenado dos valores, sem inferir que refunds são negativos.
Não faça JOIN com customers se não for necessário. Não aplique funções à
coluna created_at no WHERE. Explicite o fuso adotado e os limites inclusivo
e exclusivo, pois só foi fornecida a data, sem horário. Diferencie a janela
móvel de seis meses de seis agrupamentos mensais e sugira teste de fronteiras.

[Goal]
Entregar SQL reproduzível para a apresentação de crescimento de transações,
sem inventar números e deixando documentadas as ambiguidades temporais.
Responda com query, premissas e verificações; não alegue execução no banco.
```

## Modelo

**OpenAI — GPT-6, Codex nesta conversa**, escolhido pela geração de SQL e análise de limites temporais. Produção assistida em 02/11/2026, contexto compartilhado; sem chamada independente de API.

## Output

Premissa: como o exercício não fornece horário nem fuso, adoto **UTC** e âncora em **24/04/2026 às 00:00 UTC**. A janela é **[24/10/2025 00:00 UTC, 24/04/2026 00:00 UTC)**. Ela contém seis meses corridos e pode gerar **sete rótulos mensais**, pois outubro e abril são meses parciais. Nesse horário de âncora, abril vai até o dia 23. Se o negócio pretende incluir parte ou todo o dia 24, deve fornecer o horário de corte; os dois limites devem ser calculados a partir desse mesmo instante.

```sql
WITH parametros AS (
    SELECT TIMESTAMPTZ '2026-04-24 00:00:00+00' AS fim
), limites AS (
    SELECT
        ((fim AT TIME ZONE 'UTC') - INTERVAL '6 months')
            AT TIME ZONE 'UTC' AS inicio,
        fim
    FROM parametros
)
SELECT
    TO_CHAR(t.created_at AT TIME ZONE 'UTC', 'YYYY-MM') AS mes,
    t.category AS categoria,
    COUNT(*) AS quantidade_transacoes,
    ROUND(SUM(t.amount_cents)::numeric / 100, 2) AS volume_total_reais
FROM transactions AS t
CROSS JOIN limites AS l
WHERE t.status = 'completed'
  AND t.created_at >= l.inicio
  AND t.created_at < l.fim
GROUP BY
    TO_CHAR(t.created_at AT TIME ZONE 'UTC', 'YYYY-MM'),
    t.category
ORDER BY mes ASC, categoria ASC;
```

O cálculo do limite em horário UTC evita dependência do `TimeZone` da sessão. A conversão para reais usa `numeric`, não ponto flutuante nem divisão inteira. `ROUND(..., 2)` mantém duas casas na representação numérica usual do PostgreSQL; apresentação com vírgula é responsabilidade da camada de relatório.

Não é necessário consultar `customers`: nenhum filtro ou atributo de cliente foi solicitado. Uso `created_at` para datar a transação e `completed` como filtro de elegibilidade; trocar por `completed_at` muda a pergunta de negócio. Categorias sem movimento ficam ausentes, pois não foi pedida uma grade com zeros. Os sinais dos valores são preservados exatamente como armazenados.

Verificações propostas em banco de teste:

- Registro exatamente em 24/10/2025 00:00 UTC entra; um microssegundo antes fica fora.
- Registro imediatamente antes de 24/04/2026 00:00 UTC entra; exatamente no limite final fica fora.
- Status diferente de `completed` fica fora, mesmo dentro da janela.
- Valores 101 e 202 centavos no mesmo grupo resultam em 3,03 reais e duas transações.
- Valor negativo é somado com seu sinal, sem inversão adicional por categoria.
- `EXPLAIN (ANALYZE, BUFFERS)` em ambiente seguro permite avaliar os índices existentes antes de propor outro. Não forçar a criação de índice sem cardinalidade e plano de execução.

## Justificativa

**Task** define o relatório, o esquema e a data de referência.  
**Action** traduz os requisitos em filtros, agregações, conversão monetária e limites verificáveis.  
**Goal** exige uma consulta reproduzível e útil para a apresentação, com ambiguidades declaradas.

## Avaliação crítica

O SQL evita a falsa equivalência entre seis meses móveis e seis linhas mensais. O horário de corte e a escolha de `created_at` precisam de confirmação da PM antes do relatório definitivo. A consulta foi inspecionada, mas não executada em um PostgreSQL nesta sessão.
