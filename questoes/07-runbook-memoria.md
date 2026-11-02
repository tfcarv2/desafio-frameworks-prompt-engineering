# Questão 07 — Runbook para uso alto de memória

## Prompt

```text
[Role]
Atue como líder SRE que escreve procedimentos seguros para plantonistas.

[Input]
Alerta Beacon: [CRITICAL] High memory usage on Chronos API pods (>85% for
10min), cerca de quatro vezes por semana, MTTR atual de 30 a 40 minutos.
EKS, namespace production, seis réplicas; HPA min=4, max=12, CPU target=70%.
Deploy por Argo CD a partir de hvt/chronos-api. Dependências: Ledger
PostgreSQL e Reactor SQS. Métricas em /metrics, logs no Beacon, Grafana.
Ferramentas: kubectl, aws cli, argocd cli. Canal #oncall-chronos no Slack.
Escalação @chronos-core, resposta em 15 min comerciais ou 30 min fora.
Não foram fornecidos nome da aplicação Argo CD, cluster, região AWS, URL
da fila nem nomes de dashboards; não invente esses valores.

[Steps]
Escreva triagem com comandos concretos, evidência esperada em cada etapa,
checagem de OOMKilled, limites, rollout, dependências e HPA. Confirme contexto
e permissões antes de qualquer mutação. Diferencie working set, RSS e limite
de memória, e explique por que um HPA de CPU não resolve necessariamente
pressão de memória. Ofereça mitigação condicional com aprovação do comando
do incidente, rollback via GitOps e orçamento global de conexões do banco.
Não proponha reiniciar todos os pods ou aumentar limits/pool às cegas.

[Expectation]
Entregue runbook ponta a ponta com passos numerados, comandos de diagnóstico,
verificações, critérios objetivos de escalação e encerramento, comunicação
e ações preventivas. Marque thresholds adicionais como propostas locais,
nunca como SLA já informado. Não execute comandos nem envie mensagens.
```

## Modelo

**OpenAI — GPT-6, Codex nesta conversa**, escolhido pela organização procedural e análise de riscos operacionais. Produção assistida em 02/11/2026, contexto compartilhado; sem chamada independente de API.

## Output

### Runbook — Chronos API / memória alta

**Gatilho:** uso de memória acima de 85% por 10 minutos. **Responsável inicial:** plantonista. **Registro:** incidente e thread em `#oncall-chronos`. Os tempos abaixo são um orçamento proposto de resposta, não um SLA adicional contratado.

**1. Confirmar alvo e impacto — minutos 0–3**

```bash
kubectl config current-context
kubectl auth can-i get pods -n production
kubectl -n production get deployment chronos-api
kubectl -n production get pods -l app=chronos-api -o wide
kubectl -n production get hpa
```

Esperado: contexto corresponde ao EKS de produção no inventário, permissão de leitura, deployment encontrado e réplicas prontas próximas ao estado desejado. A referência de seis réplicas pode ter mudado pelo HPA. Se o contexto não for o correto, **parar** e obter o contexto aprovado. No Grafana/Beacon, correlacionar disponibilidade, p99, erros e tráfego na janela do alerta. Registrar início do impacto e do alerta separadamente.

**2. Confirmar a métrica e identificar os pods afetados — minutos 3–6**

```bash
kubectl -n production top pods -l app=chronos-api --containers
kubectl -n production get deployment chronos-api -o yaml
kubectl -n production get hpa -o yaml
kubectl -n production describe deployment chronos-api
```

Esperado: identificar container, request, limit, quantidade de pods, imagem atual e HPA responsável. Confirmar no dashboard a expressão do alerta e seu denominador: request, limit ou capacidade do nó. `kubectl top` não substitui essa definição. Comparar working set, RSS, cache, taxa de crescimento e tráfego por pod. Memória alta estável por cache pode ter interpretação diferente de crescimento persistente até o limite. Sem metrics-server, `top` pode falhar: registrar falha e usar as métricas já centralizadas, sem instalar componentes durante o incidente.

**3. Investigar reinícios, OOM e pressão nos nós — minutos 6–10**

```bash
kubectl -n production get events --sort-by=.metadata.creationTimestamp
kubectl -n production get pods -l app=chronos-api \
  -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,RESTARTS:.status.containerStatuses[*].restartCount,LAST_REASON:.status.containerStatuses[*].lastState.terminated.reason'
```

Selecionar **um nome real** obtido acima; não executar placeholders literalmente:

```bash
POD='<pod-confirmado>'
kubectl -n production describe pod "$POD"
kubectl -n production logs "$POD" --all-containers=true --since=20m --tail=300
# Se houve reinício, identificar primeiro o container:
kubectl -n production get pod "$POD" -o jsonpath='{.spec.containers[*].name}'
CONTAINER='<container-confirmado>'
kubectl -n production logs "$POD" -c "$CONTAINER" --previous --tail=200
NODE=$(kubectl -n production get pod "$POD" -o jsonpath='{.spec.nodeName}')
kubectl describe node "$NODE"
```

Esperado: evidência de `OOMKilled`, `Evicted`, `MemoryPressure`, reinícios e distribuição entre nós. `--previous` só funciona quando há instância anterior disponível. Não tratar ausência de log como prova de ausência de falha. Consultar Beacon para retenção maior e correlação por versão. Evitar heap dumps sem processo aprovado: podem conter dados e agravar consumo de disco/memória.

**4. Correlacionar com deploy e dependências — em paralelo até minuto 10**

```bash
kubectl -n production rollout history deployment/chronos-api
argocd app list
# Selecionar a aplicação cujo repo é hvt/chronos-api:
ARGO_APP='<nome-confirmado-na-lista>'
argocd app get "$ARGO_APP"
argocd app history "$ARGO_APP"
```

Esperado: imagem/revisão, horário de mudança e estado de sync conhecidos. No Ledger, verificar conexões usadas/livres, latência, locks e timeouts pelo monitoramento autorizado. No Reactor, conferir backlog e idade da mensagem mais antiga. Com URL de fila e região **confirmadas no inventário**:

```bash
QUEUE_URL='<url-confirmada>'
AWS_REGION='<regiao-confirmada>'
aws sqs get-queue-attributes --region "$AWS_REGION" \
  --queue-url "$QUEUE_URL" \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
```

Esperado: contadores aproximados de visíveis/em processamento e tendência no Beacon; a idade da mensagem mais antiga deve vir de métricas SQS, não desses atributos. Ausência de CPU alta não elimina vazamento, fila interna ou retenção de objetos durante espera do banco.

**5. Escolher mitigação condicionada à evidência**

| Evidência | Mitigação proposta | Verificação e limite |
|---|---|---|
| Aumento de memória começou com revisão recente | Reverter commit/configuração via fluxo GitOps aprovado, após verificar compatibilidade de schema e mensagens | Argo CD sincronizado com revisão aprovada; rollout e métricas melhorando. |
| Mais tráfego, memória por pod reduz quando a carga é distribuída, banco tem folga | Ajustar capacidade dentro do orçamento, por GitOps/HPA | Checar nodes, custo e total de conexões; HPA baseado em CPU pode não agir sobre memória. |
| Limite muito baixo, working set estável e carga testada cabe nos nós | Propor ajuste de request/limit por GitOps | Validar capacidade dos nós e custo; subir limite sem diagnóstico só adia possível OOM. |
| Crescimento persistente sem estabilização | Escalar para investigação de leak; avaliar redução de carga/feature flag existente | Preservar evidência; não reiniciar toda a frota. Reciclagem controlada é temporária e exige capacidade disponível. |
| Ledger saturado ou timeouts com fila interna crescendo | Reduzir concorrência/admitir menos trabalho por mecanismos já existentes | Evitar ampliar pool ou pods que agravem o limite de conexões. |

Antes de mutação, registrar decisão e autorização do comandante, comando/commit, evidência e critério de reversão. Mudanças emergenciais imperativas só pelo procedimento break-glass aprovado, reconciliadas com Git depois; o Argo CD pode desfazê-las. Após a mudança autorizada:

```bash
kubectl -n production rollout status deployment/chronos-api --timeout=180s
kubectl -n production top pods -l app=chronos-api --containers
kubectl -n production get pods -l app=chronos-api -o wide
argocd app get "$ARGO_APP"
```

**6. Escalar com critérios objetivos**

- **Imediatamente** se houver indisponibilidade, OOMs recorrentes, perda de réplicas prontas ou risco de perda/corrupção de dados.
- **Até 10 minutos de triagem** se a causa continuar incerta, se o crescimento persistir ou se a mitigação depender de mudança em código, banco ou recursos fora da autonomia do plantão.
- Proposta adicional: escalar se erros excederem 1% por 5 minutos, p99 violar o SLO vigente ou HPA chegar a 12 sem recuperação. Ajustar esse limiar com o time; ele não foi fornecido no cenário.
- Acionar `@chronos-core` em `#oncall-chronos` com timeline, versão, pods, gráficos, OOMs, dependências e ações já tentadas. O prazo de resposta informado é **15 minutos em horário comercial / 30 fora**, contado do acionamento. Sem resposta no prazo, seguir o contato alternativo do inventário de plantão; não inventar um nome.

**7. Encerrar e prevenir recorrência**

Proposta de encerramento a pactuar com SRE: 30 minutos sem novo OOM/restart inesperado, memória abaixo de 80% do **mesmo denominador usado no alerta**, tendência estável, réplicas desejadas prontas, erros e p99 dentro dos SLOs, Ledger sem saturação e backlog do Reactor estável/caindo dentro da meta operacional. O alerta silenciado não é prova de recuperação. Se não houver SLO definido, registrar a lacuna e obter aceite explícito do responsável técnico.

Registrar impacto, evidências, revisão aplicada, recuperação observada e risco residual. Comunicar encerramento no canal, sem enviar automaticamente por este runbook. Abrir ações para investigar leak, validar HPA por métricas adequadas, revisar recursos/probes e medir evolução da frequência semanal e do MTTR a partir da referência de 30–40 minutos.

## Justificativa

**Role** define autoria SRE orientada ao plantão. **Input** reúne alerta, infraestrutura e escalação disponíveis.  
**Steps** exige uma sequência executável com evidências e mitigação condicional.  
**Expectation** estabelece critérios verificáveis de escalação, encerramento e comunicação.

## Avaliação crítica

O runbook usa comandos de leitura antes de propor mutações e não confunde HPA de CPU com controle de memória. Faltam nomes reais de cluster, aplicação Argo, dashboards e SLOs para transformá-lo em procedimento operacional aprovado. Os placeholders estão sinalizados e nenhum comando foi executado no ambiente fictício.
