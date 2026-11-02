# Frameworks de Prompt Engineering — Hill Valley Tech

Projeto acadêmico da **Pós AIOps e IA na Engenharia de Cloud**, DevOps Pro.

**Autor:** Thiago Fernandes de Carvalho. **Registro:** 02/11/2026.

## Situação da entrega

As oito questões contêm prompt, identificação do modelo, output e justificativa. Os outputs foram produzidos pelo **GPT-6, no assistente Codex desta conversa**, a pedido do autor. São respostas geradas e registradas nesta sessão assistida; **não são capturas de oito chamadas independentes de API nem execuções isoladas em chats novos**. Os prompts documentam as instruções aplicadas a cada tarefa dentro do contexto compartilhado desta conversa; isso limita a reprodução exata dos resultados.

**Pendência do enunciado:** ele exige pelo menos **dois provedores distintos**. Esta versão usa somente **OpenAI**, conforme a preferência do autor. Antes da submissão, executar pelo menos uma questão em outro provedor, preservar a resposta original e acrescentar o nome do modelo e sua avaliação. Não declarar conformidade completa enquanto essa pendência existir.

## Questões

| Questão | Cenário | Framework | Entrega |
|---|---|---|---|
| 01 | Dockerfile do Lift | R-T-F | [Prompt, output e justificativa](questoes/01-dockerfile-lift.md) |
| 02 | Backup PostgreSQL do Ledger | R-T-F | [Prompt, output e justificativa](questoes/02-backup-ledger.md) |
| 03 | Redução de custos AWS | T-A-G | [Prompt, output e justificativa](questoes/03-custos-cloud.md) |
| 04 | Relatório SQL de transações | T-A-G | [Prompt, output e justificativa](questoes/04-transacoes-ledger.md) |
| 05 | Deployment Kubernetes legado | B-A-B | [Prompt, output e justificativa](questoes/05-deployment-chronos.md) |
| 06 | Módulo Terraform S3 | C-A-R-E | [Prompt, output e justificativa](questoes/06-terraform-s3.md) |
| 07 | Runbook de memória no EKS | R-I-S-E | [Prompt, output e justificativa](questoes/07-runbook-memoria.md) |
| 08 | Análise de incidente em andamento | R-I-S-E, escolhido | [Prompt, output e comparação](questoes/08-postmortem.md) |

## Organização e versionamento

Escolhi **um Markdown por questão**, mantendo próximos o prompt, o output, o modelo e a justificativa. Assim, o avaliador lê cada experimento sem alternar entre pastas, e o histórico do Git evidencia mudanças na instrução e no resultado. Esta decisão inicial fica registrada para comparação com as práticas de versionamento de prompts do Capítulo 4.

Os códigos estão nos blocos de output de cada questão, prontos para copiar para os arquivos indicados. A data dos experimentos é a data do registro; **24/04/2026 permanece como a data fictícia das questões 04 e 08**. Não foi substituída pela data atual.

Uma nova tentativa deve preservar a anterior, por exemplo `questoes/08-postmortem-v2.md`, com prompt exato, provedor, nome exibido do modelo, data, output sem edição e comentário sobre o que melhorou ou piorou. Correções humanas devem aparecer separadas do output original.

## Como estudar e reproduzir

1. Abra a questão e copie integralmente o bloco **Prompt**.
2. Cole em um chat novo, anote o provedor e o modelo exibido. A saída pode variar.
3. Salve a resposta real, inclusive erros, como nova tentativa, sem atribuir respostas a modelos não executados.
4. Compare com a avaliação crítica e valide o código em ambiente de teste antes de usá-lo.
5. Execute ao menos uma questão em outro provedor e atualize a situação acima antes de enviar o link.

Não são necessárias chaves de API para ler este projeto. Os cenários são fictícios; nenhum acesso à AWS, banco, cluster ou sistema real foi realizado.

## Validação

Consulte [VALIDACAO.md](VALIDACAO.md) para os checks realizados e suas limitações. Os exemplos de infraestrutura não foram aplicados em produção. Docker, Terraform, PostgreSQL e cluster Kubernetes não estavam disponíveis para testes de integração nesta sessão.

## Referências técnicas

- [PostgreSQL — funções de data e hora](https://www.postgresql.org/docs/current/functions-datetime.html)
- [Kubernetes — security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [AWS provider — S3 bucket logging](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_logging)
- [AWS — habilitar server access logging do S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/enable-server-access-logging.html)
- [Docker — boas práticas para Dockerfiles](https://docs.docker.com/build/building/best-practices/)

O material é uma resolução assistida para estudo. O autor deve revisar as escolhas e conseguir explicá-las na avaliação.
