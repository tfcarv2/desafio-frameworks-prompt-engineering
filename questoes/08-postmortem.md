# Questão 08 — Postmortem técnico e decisão durante incidente

## Framework escolhido

**R-I-S-E**: o cenário combina pressão de tempo, muitos artefatos e uma decisão operacional que exige separar evidência, hipótese e risco.

## Prompt

```text
[Role]
Atue como incident commander SRE e analista de confiabilidade. Produza em
português uma análise técnica preliminar para Doc Brown decidir em 20 minutos.
O incidente ainda está em andamento: chame o documento de postmortem
preliminar / relatório de incidente. Não invente recuperação nem causa raiz.

[Input]
Hill Valley Tech. Deploy Chronos v2.47.0 -> v2.48.0 em 2026-04-23,
Argo CD sync 18:42:11 UTC. Changelog: endpoint POST /v2/transactions/batch;
cliente Ledger com pool movido para nova biblioteca interna; psycopg
3.1.18 -> 3.2.0; timeout do Ledger de 5s para 2s.
Métricas em 2026-04-24 (UTC):
hora,p99_latency_ms,req_rate_s,err_rate_pct
13:30,420,1200,0.2
13:45,510,1450,0.3
14:00,780,1780,0.8
14:10,2400,2100,4.5
14:15,5200,2400,8.2
14:20,8100,2650,11.7
Logs do pod chronos-api-79c4d8b9-xk2jp:
14:19:48 [ERROR] [ledger-client] connection pool exhausted (max=20, active=20, waiting=147)
14:19:49 [WARN] [ledger-client] query timeout after 2000ms: SELECT ... FROM transactions WHERE ...
14:19:49 [ERROR] [handler] POST /v2/transactions/batch failed: context deadline exceeded
14:19:50 [ERROR] [ledger-client] connection reset by peer
14:19:51 [WARN] [circuit-breaker] ledger-client OPEN (threshold 50%, current 87%)
14:19:52 [ERROR] [reactor] failed to publish message: chronos-api upstream error
Reactor chronos-transactions: 50.127 mensagens acumuladas; crescimento
aproximado de 800/min; consumer lag de 18 minutos e aumentando.
Cluster: 12/12 pods running (HPA no máximo); CPU média 62%; memória 71%.
Ledger: 240 conexões ativas de limite RDS 250.
Alternativas apresentadas: rollback v2.48.0 ou scaling emergencial
(aumentar limits do RDS e pool de conexões).

[Steps]
1. Construa timeline com os horários fornecidos e registre que a série cobre
50 minutos, embora tenha sido descrita como últimos 30 minutos.
2. Calcule evolução de tráfego, p99, erros e ocupação de conexões. Distinga
os 87% do circuito local dos 11,7% de erros globais.
3. Separe fatos, hipóteses classificadas por confiança e evidências faltantes.
Explique a hipótese de 12 x 20 = 240 sem afirmá-la como configuração global
comprovada; workers e outros clientes podem mudar esse cálculo.
4. Compare rollback e scaling por reversibilidade, risco e pré-requisitos.
Não aumente pool apenas porque há fila; banco perto do limite pode piorar.
5. Recomende uma ação condicionada a gates objetivos, com plano para 0-5,
5-10 e 10-20 minutos, validação e critérios de interrupção/escalonamento.
Rollback exige confirmar migrações, compatibilidade de schema/mensagens e
impacto do endpoint novo. Respeite Argo CD/GitOps. Não execute nenhuma ação.
6. Sugira ações posteriores com responsável por função e critério de pronto.

[Expectation]
Entregue resumo executivo, timeline, fatos e hipóteses, matriz de decisão,
plano de 20 minutos, riscos e ações preventivas. Não atribua culpa, não estime
transações perdidas como fato e não declare causa raiz ou recuperação sem
evidência. Limiares de recuperação adicionais devem ser propostas explícitas.
```

## Modelo

**OpenAI — GPT-6, Codex nesta conversa**, escolhido pela síntese de evidências, cálculos e comparação de decisões operacionais. Produção assistida em 02/11/2026, contexto compartilhado; sem chamada independente de API.

## Output

### Postmortem preliminar — Chronos / Ledger / Reactor

**Situação às 14:20 UTC de 24/04/2026: incidente em andamento.** Não há horário de recuperação nem causa raiz confirmada. A evidência mais forte é contenção no caminho de acesso ao Ledger: pool local esgotado, espera de 147 requisições, timeouts e banco com **96% das conexões ocupadas**. CPU e memória médias dos pods não indicam, por si, saturação de compute. `Running` não comprova readiness nem sucesso de requisições.

**Recomendação:** priorizar a avaliação rápida de um **rollback controlado para v2.47.0**, condicionado à compatibilidade de schema, mensagens e uso do endpoint novo. Não aumentar o pool neste momento: há apenas **10 conexões de folga** no limite informado. Escalar o RDS só deve seguir com evidência de capacidade insuficiente, orçamento global de conexões e plano de mudança/reversão; aumentar `max_connections` sozinho não cria CPU, memória ou I/O.

### Timeline conhecida

| UTC | Evidência |
|---|---|
| 23/04 18:42:11 | Argo CD sincroniza v2.48.0; mudanças em batch, cliente/pool, driver e timeout. |
| 24/04 13:30 | 1.200 req/s, p99 420 ms, erros 0,2%. |
| 13:45 | 1.450 req/s, p99 510 ms, erros 0,3%. |
| 14:00 | 1.780 req/s, p99 780 ms, erros 0,8%. |
| 14:10 | 2.100 req/s, p99 2.400 ms, erros 4,5%. |
| 14:15 | 2.400 req/s, p99 5.200 ms, erros 8,2%. |
| 14:19:48–52 | Pool esgotado, timeout 2s, batch falha, reset de conexão, circuito abre e publicação no Reactor falha. |
| 14:20 | 2.650 req/s, p99 8.100 ms, erros 11,7%; 12 pods; banco 240/250; backlog 50.127. |

Os pontos de métricas cobrem **50 minutos**, não 30. A proximidade temporal dos logs ajuda na correlação, mas sua ordem não prova toda a cadeia causal. O deploy ocorreu cerca de 19h38 antes do último ponto: uma regressão acionada por carga é plausível, mas não comprovada por proximidade imediata.

### Quantificação e fatos

- Tráfego: 1.200 → 2.650 req/s, **+120,83%**, ou 2,21 vezes.
- p99: 420 → 8.100 ms, **19,29 vezes**; aumento de aproximadamente 1.828,57%.
- Erros globais: 0,2% → 11,7%, **+11,5 pontos percentuais**, ou 58,5 vezes.
- No ponto final, 2.650 × 11,7% equivale a cerca de **310 requisições com erro por segundo**, se taxa e percentual tiverem a mesma janela/denominador. Isso não mede perda definitiva de transações.
- Ledger: **240/250 = 96%**; somente 10 conexões de margem.
- `12 × 20 = 240` é compatível com um pool de 20 por pod, mas é **hipótese**: o log é de um pod; podem existir pools por processo, workers e outros clientes.
- O circuito com 87% refere-se ao cliente/escopo e janela locais; não equivale à taxa global de erro de 11,7%.
- O backlog e sua idade crescem. A falha de publicação observada exige reconciliar dados; não prova que todas as 50.127 mensagens têm essa origem nem que foram perdidas.

### Hipóteses e verificações

| Hipótese | Confiança atual | Evidência a favor | O que falta |
|---|---|---|---|
| Contenção de conexões é gargalo imediato relevante | Alta | Pool esgotado, espera, 240/250 e timeouts | Sessões por origem, tempo de checkout, locks e queries ativas. |
| Nova biblioteca alterou dimensionamento ou liberação de conexões | Média | Mudança recente exatamente no pool; sintoma compatível | Diff de configuração/código, pools por processo e sessões idle in transaction. |
| Endpoint batch amplificou trabalho no Ledger | Média | Endpoint novo aparece em erro durante pico | Mix de tráfego, tamanho dos lotes, queries por request e comparação entre versões. |
| Timeout menor agravou erros e eventuais retries | Média | 5s → 2s; log de timeout em 2s | Política real de retry/cancelamento e duração de queries; retries não estão comprovados. |
| RDS limitado por CPU/I/O/locks | Indeterminada | Pode prolongar retenção de conexões | CPU, memória, IOPS, wait events, locks e plano de consultas. |
| Regressão específica do psycopg | Baixa | Houve upgrade | Reprodução e evidência específica; versão sozinha não estabelece causalidade. |

### Matriz de decisão

| Alternativa | Benefício esperado | Riscos / gates | Reversibilidade |
|---|---|---|---|
| Rollback para v2.47.0 | Remove simultaneamente mudanças suspeitas do caminho crítico | Confirmar migrações reversíveis, schema/mensagens compatíveis, clientes dependentes de `/batch`, versão anterior saudável e capacidade para a carga atual | Geralmente rápida pelo commit/artefato anterior; não reverte efeitos em dados. |
| Aumentar pool | Pode diminuir espera local se houver capacidade real no banco | Com 96% de conexões, pode esgotar o RDS e piorar concorrência/latência; requer orçamento por processo/pod e por cliente | Configuração pode ser revertida, mas sessões e pressão persistem até drenagem. |
| Escalar RDS | Pode ajudar se CPU/memória/I/O limitarem o banco | Não há essas métricas; resize/parâmetros podem envolver interrupção/reboot e tempo maior que 20 min | Retorno pode ser lento e disruptivo; Multi-AZ não garante ausência de impacto. |
| Reduzir trabalho admitido por controle já existente | Alivia pressão enquanto a investigação segue | Requer mecanismo disponível e aprovado, política de retry/idempotência e avaliação do impacto nos clientes | Em geral rápida se houver feature flag/rate limit já implantado. |

### Plano proposto de 20 minutos

**0–5 min:** comandante registra incidente, designa SRE/DBA/aplicação e preserva gráficos/logs. Validar a saúde real dos pods, sessões por cliente, waits/locks, configuração do pool, diff do deploy e status no Argo CD. Confirmar compatibilidade de rollback e demanda do endpoint batch com os responsáveis. Não disparar reinícios em massa nem aumentar conexões.

**5–10 min:** se gates de compatibilidade forem confirmados, registrar aprovação e reverter a referência da imagem/configuração no GitOps para a revisão anterior aprovada, acompanhando a sincronização Argo CD. Não usar `rollout undo` solto para ser desfeito pelo reconciliador. Considerar que rolling update com surge pode aumentar temporariamente conexões; ajustar estratégia conforme capacidade e plano aprovado. Se rollback não for seguro, pausar essa opção e aplicar somente contenção já disponível/aprovada enquanto DBA avalia capacidade do RDS.

**10–20 min:** observar rollout, conexões, tempo de espera do pool, erros, p99 e fila. Confirmar que a melhora decorre da alteração e não só de queda do tráfego, comparando carga e mix. Propostas de gates para recuperação inicial: conexões abaixo de 85% (no máximo 212 de 250), espera do pool caindo, erros abaixo de 1%, p99 abaixo de 1s e todos os pods desejados prontos por pelo menos 10 minutos. Esses limites são **provisórios**, a substituir pelos SLOs reais, e não autorizam encerrar antes de cumprir a janela. Backlog deve apresentar tendência de drenagem e idade decrescente, com prazo de normalização estimado por taxa real de consumo líquido.

Se surgirem incompatibilidade de dados, aumento abrupto de erros, regressão de readiness ou saturação maior, interromper a mudança e escalar imediatamente. Se o rollback não melhorar o sistema, a hipótese de regressão perde força, e a próxima decisão depende das métricas do banco e do perfil de consultas. Não executar sucessivas alterações simultâneas sem registrar causalidade e responsável.

### Pendências e ações posteriores

| Responsável por função | Ação | Critério de pronto |
|---|---|---|
| Engenharia Chronos | Testar regressão do pool e do batch | Teste de pico com checkout/retorno de conexões instrumentados e sem crescimento de sessões retidas. |
| DBA + SRE | Definir orçamento global de conexões | Somatório por pod/processo/cliente mais reserva administrativa cabe na capacidade validada, inclusive durante surge. |
| Engenharia + QA | Validar timeout, retry e idempotência | Testes demonstram cancelamento correto, ausência de duplicidade e política de backoff sob falhas. |
| SRE/Beacon | Monitorar espera no pool, conexões e fila | Dashboard/alertas testados com denominadores e janelas documentados. |
| Dono do Reactor | Reconciliar transações e publicações | Mensagens/transações identificadas e verificadas; replay somente quando idempotência e duplicidades estiverem controladas. |
| Dono do incidente | Concluir postmortem após recuperação | Impacto, timeline final, evidências causais, recuperação e ações com prazos registrados e revisados sem atribuição de culpa. |

**Causa raiz: ainda não confirmada. Recuperação: ainda não observada.**

## Justificativa estendida

**Role** estabelece uma visão de comando de incidente e confiabilidade, adequada à pressão de tempo. **Input** preserva os artefatos e os números, sem resumir a ponto de perder sinais relevantes. **Steps** força a progressão de timeline para cálculos, hipóteses, comparação e decisão. **Expectation** impõe um resultado verificável e limita alegações de causa/recuperação.

| Candidato | O que se ganha | O que se perde neste cenário |
|---|---|---|
| **R-I-S-E — escolhido** | Sequência explícita para correlacionar evidências e decidir, com expectativas de qualidade e cautela operacional | Prompt maior e preparação mais trabalhosa; os passos precisam priorizar os 20 minutos. |
| **C-A-R-E** | Contexto rico, ação clara, resultado e possibilidade de fornecer um bom exemplo de relatório | Um exemplo pode ancorar indevidamente uma causa/decisão; a sequência de verificação precisaria ser adicionada dentro de Action, pois não é explícita no nome dos componentes. |
| **B-A-B** | Expõe bem o estado degradado, a recuperação desejada e a ponte de mitigação | Pode induzir a descrever o estado recuperado como conhecido e começar pela solução, quando causa e segurança do rollback ainda são incertas; ainda exigiria separação adicional de fatos e hipóteses. |

Os frameworks alternativos também poderiam funcionar com instruções bem escritas; a escolha não garante qualidade por si só. R-I-S-E foi preferido porque a estrutura do problema exige **uma investigação ordenada com decisão condicionada**, não apenas um formato de relatório ou uma transformação de estado.

## Avaliação crítica

O resultado identifica o gargalo aparente sem chamar correlação de causa raiz. O ponto mais importante é rejeitar aumento automático do pool com o banco a 96%. Faltam métricas internas do RDS, teste da versão anterior sob a carga atual e validação de compatibilidade; essas lacunas impedem recomendar rollback incondicional ou alegar recuperação.
