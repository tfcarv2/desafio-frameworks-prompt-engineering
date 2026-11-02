# Questão 06 — Módulo Terraform S3 no padrão interno

## Prompt

```text
[Context]
Você está na Hill Valley Tech. Strickland exige tags Owner, CostCenter e
Environment em recursos que suportem tags; prefixo hvt- nos nomes; buckets
S3 com SSE-S3 no mínimo, versionamento ativo, block public access total e
server access logging. Variáveis em variables.tf com description e type.
O destino de logs é um bucket central já existente, administrado por outro
time: não crie uma cadeia infinita de buckets de logs. Explicite os requisitos
de policy, conta/região e proteção desse destino antes do uso do módulo.

[Action]
Gere módulo reutilizável em Terraform com recursos separados de S3 para
criptografia, versioning, public access block e logging; tags obrigatórias não
podem ser sobrescritas por tags extras. Inclua validações de entradas,
outputs, versions.tf e exemplo de consumo. Evite ACLs legadas. Use o estilo
de locals.common_tags e merge do exemplo. Não alegue terraform apply/validate.

[Result]
Quero um módulo legível e reutilizável que produza buckets aderentes ao
padrão, com pré-requisitos explícitos, exemplo sem credenciais e orientações
para fmt, validate e plan. Não configure provider dentro do módulo filho.

[Example]
variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}
locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}
resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags = merge(local.common_tags, {
    Name = "hvt-vpc-${var.environment}"
  })
}
```

## Modelo

**OpenAI — GPT-6, Codex nesta conversa**, escolhido pela geração de IaC modular e revisão de políticas. Produção assistida em 02/11/2026, contexto compartilhado; sem chamada independente de API.

## Output

Estrutura proposta: `modules/s3/{versions,variables,main,outputs}.tf` e `examples/basic/main.tf`.

### modules/s3/versions.tf

```hcl
terraform {
  required_version = ">= 1.5.0, < 2.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0, < 7.0"
    }
  }
}
```

### modules/s3/variables.tf

```hcl
variable "name" {
  description = "Identificador de 3 a 40 caracteres; incluir sufixo globalmente único."
  type        = string
  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$", var.name))
    error_message = "Use 3 a 40 caracteres: letras minúsculas, números e hífen."
  }
}

variable "owner" {
  description = "Time responsável pelo bucket."
  type        = string
  validation {
    condition     = length(trimspace(var.owner)) > 0
    error_message = "Owner não pode ser vazio."
  }
}

variable "cost_center" {
  description = "Centro de custo responsável."
  type        = string
  validation {
    condition     = length(trimspace(var.cost_center)) > 0
    error_message = "CostCenter não pode ser vazio."
  }
}

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)."
  type        = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Ambiente deve ser dev, staging ou production."
  }
}

variable "logging_bucket_id" {
  description = "Nome do bucket central de logs existente, na mesma conta e região."
  type        = string
  validation {
    condition     = can(regex("^hvt-[a-z0-9-]+[a-z0-9]$", var.logging_bucket_id)) && length(var.logging_bucket_id) <= 63
    error_message = "Destino deve ter nome S3 válido com prefixo hvt-."
  }
}

variable "tags" {
  description = "Tags adicionais; as tags obrigatórias e Name prevalecem."
  type        = map(string)
  default     = {}
}
```

### modules/s3/main.tf

```hcl
locals {
  bucket_name = "hvt-${var.name}-${var.environment}"
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_s3_bucket" "this" {
  bucket        = local.bucket_name
  force_destroy = false
  tags = merge(var.tags, local.common_tags, {
    Name = local.bucket_name
  })
}

resource "aws_s3_bucket_ownership_controls" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  ignore_public_acls      = true
  block_public_policy     = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_logging" "this" {
  bucket        = aws_s3_bucket.this.id
  target_bucket = var.logging_bucket_id
  target_prefix = "s3-access/${local.bucket_name}/"
  lifecycle {
    precondition {
      condition     = var.logging_bucket_id != local.bucket_name
      error_message = "O bucket de origem não pode receber seus próprios access logs."
    }
  }
}

data "aws_iam_policy_document" "tls" {
  statement {
    sid     = "DenyInsecureTransport"
    effect  = "Deny"
    actions = ["s3:*"]
    resources = [
      aws_s3_bucket.this.arn,
      "${aws_s3_bucket.this.arn}/*"
    ]
    principals {
      type        = "*"
      identifiers = ["*"]
    }
    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}

resource "aws_s3_bucket_policy" "this" {
  bucket = aws_s3_bucket.this.id
  policy = data.aws_iam_policy_document.tls.json
}
```

### modules/s3/outputs.tf

```hcl
output "bucket_id" {
  description = "Nome do bucket criado."
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN do bucket criado."
  value       = aws_s3_bucket.this.arn
}

output "logging_prefix" {
  description = "Prefixo usado no bucket central de logs."
  value       = aws_s3_bucket_logging.this.target_prefix
}
```

### examples/basic/main.tf

```hcl
terraform {
  required_version = ">= 1.5.0, < 2.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0, < 7.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

module "ledger_bucket" {
  source            = "../../modules/s3"
  name              = "ledger-123456789012"
  owner             = "sre"
  cost_center       = "CC-100"
  environment       = "production"
  logging_bucket_id = "hvt-central-logs-123456789012"
  tags = {
    Project = "ledger"
  }
}
```

Os nomes e o account ID do exemplo são fictícios. S3 tem namespace global: escolher sufixo disponível. O destino de logs deve existir **antes do apply**, na mesma conta e região, com versionamento, SSE-S3, bloqueio público, tags e política de retenção aprovados. O time central deve adicionar à policy existente permissão `s3:PutObject` para `logging.s3.amazonaws.com`, restrita ao prefixo `s3-access/<nome-origem>/*`, a `aws:SourceAccount` e a `aws:SourceArn` do bucket de origem. Não sobrescrever a policy compartilhada a partir deste módulo.

O bucket central deve ter sua estratégia de auditoria aprovada separadamente; não o criar recursivamente com o mesmo módulo nem habilitar entrega de logs para si próprio. A existência de `aws_s3_bucket_logging` não prova entrega: após tráfego de teste, verificar chegada efetiva de logs considerando a natureza assíncrona/best effort do serviço.

As tags são atribuídas ao recurso de bucket, que suporta tags; configurações auxiliares de S3 não expõem necessariamente tags próprias. `merge` dá precedência às tags obrigatórias. O módulo não inclui credenciais, ACLs ou provider configurado internamente. `force_destroy=false` impede destruição automática de conteúdo pelo provider, mas não substitui controles de mudança e recuperação.

```bash
terraform fmt -recursive
cd examples/basic
terraform init
terraform validate
terraform plan
```

Executar em conta de testes autorizada, fixar a seleção do provider no lockfile do projeto consumidor e revisar o plano antes de aplicar. `fmt` e `validate` verificam sintaxe/esquema, não comprovam compliance operacional. Conferir no S3 encryption, versioning, public access block e recebimento de logs. Não houve execução de Terraform nesta sessão.

## Justificativa

**Context** fornece o padrão de compliance e a gestão externa do destino de logs.  
**Action** pede os arquivos e os controles concretos; **Result** define reutilização e validação esperadas.  
**Example** fixa o estilo de variáveis, locals e merge que orienta a resposta.

## Avaliação crítica

O módulo cobre os controles solicitados para os buckets que cria, mas a conformidade ponta a ponta depende do destino de logs externo. A faixa ampla do provider requer lockfile e validação no consumidor. Uma iteração posterior deve testar a política central e a entrega de logs em conta isolada.
