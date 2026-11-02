# Questão 03 — Redução de custos cloud

## Prompt

```text
[Task]
Produza um relatório executivo e técnico de redução de custos AWS para a CEO
da Hill Valley Tech. Meta: reduzir 15% até o fim do próximo trimestre sem
degradar SLA. Use exclusivamente este CSV como base financeira:
servico,categoria,custo_mensal_usd,uso_medio_pct,observacao
EC2 reservada,compute,4200,72,contrato de 1 ano
EC2 on-demand,compute,8200,45,workloads variaveis
EKS,compute,6700,58,3 clusters
RDS PostgreSQL,databases,8200,62,multi-AZ
ElastiCache Redis,databases,2100,40,cluster de producao
S3 Standard,storage,3100,,5 buckets principais
EBS gp3,storage,1600,68,volumes de producao
CloudWatch Logs,observability,2800,,retencao de 90 dias
CloudWatch Metrics,observability,900,,
Data Transfer Out,network,1900,,trafego entre regioes
NAT Gateway,network,1200,,3 gateways ativos
Lambda,compute,900,30,~12M invocacoes/mes

[Action]
Calcule total, meta mensal equivalente e baseline trimestral sem crescimento.
Ordene oportunidades por economia mensal estimada; para cada uma, informe
USD/mês, percentual da conta total, esforço baixo/médio/alto, riscos e
pré-requisitos. Declare explicitamente taxas de economia como hipóteses,
não fatos medidos. Não trate uso médio baixo como capacidade ociosa garantida.
Evite dupla contagem de EKS versus EC2, contratos já assumidos e propostas
que removam Multi-AZ ou observabilidade exigida. Some o cenário proposto,
compare com 15% e apresente sensibilidade e etapas de validação com rollback.

[Goal]
Dar à diretoria um plano priorizado e verificável para atingir 15% preservando
SLA. Separe potencial estimado, economia realizada e riscos. Responda com
resumo, tabela priorizada e plano por fases de até 90 dias.
```

## Modelo

**OpenAI — GPT-6, Codex nesta conversa**, escolhido pela combinação de análise quantitativa e síntese executiva. Produção assistida em 02/11/2026, contexto compartilhado; sem chamada independente de API.

## Output

O total informado é **US$ 41.800/mês**. A meta de 15% equivale a **US$ 6.270/mês**, com gasto final de **US$ 35.530/mês**. Sem crescimento, o baseline de três meses é **US$ 125.400**. Uma redução de 15% já vigente durante todo o trimestre pouparia US$ 18.810; alcançar esse patamar apenas ao final do trimestre não produz automaticamente essa economia acumulada.

O cenário abaixo apresenta **hipóteses de planejamento**, a validar com billing detalhado e testes. Percentual da conta = economia mensal / 41.800 × 100. Médias de utilização não determinam sozinhas capacidade removível.

| Prioridade | Oportunidade e hipótese | US$/mês | % da conta | Esforço | Riscos e pré-requisitos |
|---|---|---:|---:|---|---|
| 1 | EC2 on-demand: 20% com rightsizing e horários de workloads elegíveis | 1.640 | 3,92% | Médio | Ver picos, memória, I/O e SLA; testar canário. Compromissos só após medir demanda estável. |
| 2 | CloudWatch Logs: 40% com classes/retenção e redução de logs redundantes | 1.120 | 2,68% | Médio | Aprovação de compliance; preservar logs de auditoria e capacidade de investigação; retenção menor só quando permitida. |
| 3 | EKS: 15% com requests e consolidação de capacidade | 1.005 | 2,40% | Alto | Reconciliar essa linha com EC2 antes de somar; avaliar PDB, disponibilidade por zona e autoscaling. |
| 4 | RDS: 10% com tuning e dimensionamento validado | 820 | 1,96% | Alto | Manter Multi-AZ; medir conexões, IOPS, memória e latência, inclusive no pico; planejar reversão. |
| 5 | S3: 15% com Lifecycle e classes elegíveis | 465 | 1,11% | Médio | Medir padrões de acesso; incluir custos de transição, recuperação e permanência mínima. |
| 6 | Redis: 20% com dimensionamento compatível com working set | 420 | 1,00% | Médio | Avaliar evictions, cache hit, failover e latência; impacto no RDS se diminuir cache. |
| 7 | Transferência: 20% reduzindo tráfego inter-regional desnecessário | 380 | 0,91% | Alto | Confirmar SKU/caminho cobrado; manter DR e requisitos geográficos. |
| 8 | NAT: 15% com endpoints economicamente viáveis e rotas | 180 | 0,43% | Médio | Comparar custo total de endpoints; manter resiliência por zona; separar economia de transferência já contabilizada. |
| 9 | EBS gp3: 10% removendo sobras e ajustando IOPS/throughput | 160 | 0,38% | Médio | Conferir anexos, capacidade, snapshots e performance; não apagar volumes por aparente inatividade. |
| 10 | CloudWatch Metrics: 10% reduzindo séries sem uso | 90 | 0,22% | Baixo | Inventariar dashboards e alarmes; não remover sinais do SLA. |
| 11 | Lambda: 10% por ajuste de memória/duração | 90 | 0,22% | Médio | Benchmark com percentis de latência e cold start; número de invocações sozinho não define economia. |
| — | EC2 reservada: sem economia adicional incluída | 0 | 0,00% | — | Contrato de um ano permanece; melhorar cobertura não elimina pagamento contratado. |
| | **Total das hipóteses** | **6.370** | **15,24%** | | **Somente se as linhas forem independentes e os ganhos comprovados.** |

Gasto projetado nesse cenário: **US$ 35.430/mês**. A margem sobre a meta é apenas **US$ 100/mês**, insuficiente para tratar o objetivo como assegurado. Se só metade das hipóteses se confirmar, a economia será US$ 3.185/mês, ou **7,62%**. Se a linha EKS duplicar custos EC2 e sua oportunidade for excluída, o potencial cai a US$ 5.365, **12,83%**. Portanto, ainda não existe evidência para prometer 15%.

Plano de execução:

1. **Dias 1–15 — medir e reconciliar:** FinOps e donos dos serviços confirmam custos sem sobreposição, sazonalidade, descontos e compromissos. SRE registra baseline de disponibilidade, p95/p99, erros e saturação. Compliance valida retenções.
2. **Dias 16–45 — mudanças de menor risco:** reduzir telemetria redundante aprovada, tratar recursos comprovadamente sem uso e pilotar Lifecycle. Direitos de exclusão exigem identificação do dono e política de recuperação. Apurar economia líquida, incluindo novos custos.
3. **Dias 46–75 — capacidade e arquitetura:** pilotos em EC2/EKS/Redis/RDS, um por vez para atribuir impacto. Manter redundância, testar pico e failover e comparar com baseline. Cada mudança tem dono e configuração anterior versionada.
4. **Dias 76–90 — comprovar:** comparar custo normalizado por volume de transações e gasto absoluto, distinguir efeito de demanda e de otimização. Só classificar como realizada a economia observada. Se a projeção ficar abaixo de 15%, rever escopo e prazo com a diretoria.

Critério de continuidade: nenhuma quebra dos SLOs vigentes. Se percentis de latência, erros, capacidade de recuperação ou disponibilidade piorarem além do orçamento aprovado, reverter a mudança afetada. Não há valores de SLO no CSV: eles devem vir do contrato operacional, não ser inventados neste relatório.

## Justificativa

**Task** identifica o relatório, os dados e a restrição de SLA.  
**Action** transforma a demanda em cálculos, priorização e análise de risco verificáveis.  
**Goal** conecta as recomendações à meta de 15% e à decisão executiva sem prometer ganhos não medidos.

## Avaliação crítica

As contas fecham, mas os percentuais de economia são hipóteses. O principal risco é somar custos de EKS e EC2 que já se sobreponham. Uma próxima iteração usaria CUR/Cost Explorer e séries de pico para substituir hipóteses por estimativas fundamentadas.
