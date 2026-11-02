# Validação e limites

Registro de 02/11/2026. A avaliação de código foi feita sobre os blocos de output extraídos dos documentos, sem acessar infraestrutura externa.

| Verificação | Resultado | O que comprova |
|---|---|---|
| Oito documentos, campos Prompt/Modelo/Output/Justificativa e cercas Markdown balanceadas | Aprovado | Estrutura local das oito entregas. |
| `bash -n` no script da Q02 | Aprovado | Sintaxe Bash; não comprova backup ou permissões. |
| Compilação dos dois heredocs Python da Q02 | Aprovado | Sintaxe Python; não realiza exclusões no S3. |
| Parse YAML dos dois documentos da Q05 | Aprovado | YAML válido com Deployment e PDB; não é validação pela API Kubernetes. |
| Recalcular custos Q03 | Aprovado | Total 41.800; meta 6.270; hipóteses 6.370; sem EKS 5.365 USD/mês. |
| Recalcular percentuais Q08 | Aprovado | Conexões 96%; tráfego +120,83%; p99 19,29 vezes; erros +11,5 p.p. |
| Docker build / smoke test | Não executado | Código real do Lift e Docker não disponíveis. |
| Terraform fmt / validate / plan | Não executado | Binário Terraform e conta AWS de teste não disponíveis. |
| SQL em PostgreSQL e EXPLAIN | Não executado | Banco e cliente PostgreSQL não disponíveis. |
| Restauração de backup / testes S3 | Não executado | Nenhuma credencial ou infraestrutura do cenário fictício foi acessada. |
| Aplicação de manifests / comandos de runbook | Não executado | Nenhum cluster acessado. |
| Dois provedores de IA | **Pendente** | Todas as respostas desta versão usam OpenAI GPT, conforme preferência do autor. |

## Proveniência dos outputs

As respostas foram elaboradas e registradas pelo GPT-6, assistente Codex nesta conversa. Os prompts descrevem as instruções aplicadas às tarefas dentro da sessão compartilhada. Não houve oito chamadas separadas de API, nem transcrições importadas de sessões isoladas. Não são atribuídos outputs ao Gemini, Claude ou a outro provedor que não tenha sido executado.

Para uma reprodução experimental mais controlada, usar um chat novo por questão e registrar a resposta real junto do nome de modelo exibido, data e eventual identificador público de conversa, se o autor quiser compartilhá-lo. Não publicar tokens, chaves de API ou conversas contendo dados privados.

## Próximo passo antes da submissão

Executar pelo menos uma questão em um segundo provedor, preservar a resposta como nova tentativa, acrescentar a justificativa de escolha do modelo e comparar acertos/erros. Atualizar README e esta tabela apenas após a execução. O repositório atual é uma resolução assistida útil para estudo, mas ainda não cumpre essa exigência do enunciado.
