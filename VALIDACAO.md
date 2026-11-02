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
