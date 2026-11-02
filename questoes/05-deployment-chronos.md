# Questão 05 — Modernizar deployment legado

## Prompt

```text
[Before]
O Chronos tem este Deployment legado no namespace production:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"

[After]
Quero uma base de produção com alta disponibilidade, imagem versionada,
secretKeyRef para segredo previamente provisionado, requests/limits, readiness
e liveness, não-root, filesystem read-only com /tmp gravável, capabilities
removidas, seccomp e token de service account desabilitado se desnecessário.
Inclua rollout seguro, distribuição entre nós e zonas e PodDisruptionBudget.
Não reproduza os valores sensíveis no resultado. Imagem e caminhos de health
check não foram fornecidos: trate-os como exemplos a validar, não como fatos.

[Bridge]
Transforme o YAML preservando nome, namespace, seletor e porta 8080. Entregue
Deployment e PDB; explique escolhas e passos para migrar e validar com GitOps.
Não crie Secrets com valores de exemplo em base64. Defina limites de recursos
como hipótese inicial. Diferencie disponibilidade desejada de garantida,
compatibilidade da aplicação e dependências da infraestrutura. Não aplique nada.
```

## Modelo

**OpenAI — GPT-6, Codex nesta conversa**, escolhido pela transformação e revisão de manifests Kubernetes. Produção assistida em 02/11/2026, contexto compartilhado; sem chamada independente de API.

## Output

O exemplo usa **três réplicas** e a tag **v2.48.0 como valor ilustrativo**, não como versão aprovada para implantação. Esta questão é independente da Q08: nela, v2.48.0 está sob investigação. Substituir pela imagem aprovada no registry real e, preferencialmente, fixar também o digest. `/health/live` e `/health/ready` são **contratos propostos**; confirmar ou implementar antes de usar as probes.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
spec:
  replicas: 3
  revisionHistoryLimit: 5
  minReadySeconds: 10
  progressDeadlineSeconds: 300
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 45
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: chronos-api
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: chronos-api
      containers:
        - name: api
          image: chronos-api:v2.48.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: chronos-api-runtime
                  key: DB_PASSWORD
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: chronos-api-runtime
                  key: JWT_SECRET
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: [ALL]
          startupProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 5
            failureThreshold: 30
            timeoutSeconds: 2
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 2
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            periodSeconds: 5
            failureThreshold: 3
            timeoutSeconds: 2
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 128Mi
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chronos-api
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: chronos-api
```

As três réplicas não garantem HA por si só: é necessário ter nós, zonas, capacidade de surge, balanceamento e dependências disponíveis. As constraints distribuem pods entre os domínios elegíveis existentes; elas não criam nós nem obrigam a existência de várias zonas. Validar que o node group realmente cobre múltiplas zonas. O PDB atua em disrupções voluntárias; não impede perda de nó nem controla o rollout do Deployment.

Passos de migração:

1. Provisionar `chronos-api-runtime` por um controlador de segredos ou mecanismo aprovado e rotacionar os valores expostos no legado. Remover texto do manifesto não apaga segredos do histórico Git.
2. Confirmar imagem, UID/GID, rotas de saúde, arquivos gravados pela aplicação e shutdown gracioso. Liveness deve refletir saúde do processo; não reiniciar todos os pods por indisponibilidade temporária do banco.
3. Ajustar recursos com métricas e teste de pico. Os números acima são pontos de partida, não dimensionamento medido. Conferir eventual HPA: o controlador pode alterar réplicas, e GitOps deve respeitar isso.
4. Versionar via PR, validar schema e `kubectl apply --dry-run=server -f chronos.yaml` em cluster de homologação. Confirmar permissões de RBAC e segredo existente sem imprimir seu conteúdo.
5. Implantar por GitOps, acompanhar rollout, readiness, erros, p99 e distribuição. Se regredir, reverter o commit para a imagem e configuração aprovadas. Não usar mudança imperativa que o reconciliador vá desfazer.

NetworkPolicy, Service, ingress, imagePullSecrets, HPA e integração de segredos dependem da arquitetura real. Não são inventados neste manifest.

## Justificativa

**Before** mostra o estado legado e seus problemas concretos.  
**After** descreve propriedades observáveis da versão desejada.  
**Bridge** pede a transformação e o caminho de migração/validação, conectando os dois estados sem esconder premissas.

## Avaliação crítica

O resultado remove credenciais literais e acrescenta mecanismos de segurança e disponibilidade. Imagem, endpoints, UID e recursos são premissas que impedem uso direto em produção. Uma nova iteração deveria receber o contrato real de saúde e os perfis de consumo para eliminar esses exemplos.
