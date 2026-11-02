# Questão 02 — Backup do Ledger

## Prompt

```text
[Role]
Atue como SRE especialista em PostgreSQL, Bash no Ubuntu 22.04 e backups AWS.

[Task]
Gere um script Bash diário para ledger-db.internal.hvt.io:5432, banco
ledger_prod, usuário backup_user. PGPASSWORD já chega no ambiente por um
mecanismo existente integrado ao AWS Secrets Manager e à IAM role da EC2;
não implemente busca de um SecretId desconhecido nem imprima a senha.
Região us-east-1; diretório /var/backups/ledger com 80 GB livres; dump gzip
estimado em 12 GB. Faça pg_dump | gzip com pipefail, envie por aws s3 cp
ao bucket hvt-ledger-backups, retenha 30 dias no S3, registre timestamps em
/var/log/ledger-backup.log e retorne códigos de saída úteis em falhas.
Evite execução concorrente, arquivos parciais, apagamento fora do prefixo
ledger/ e exclusão remota antes de confirmar upload. A retenção deve usar
LastModified em UTC, tratar paginação e verificar falhas de delete-object.
Use credenciais AWS da role, não chaves estáticas. Explique requisitos de
permissão no SO/IAM, autenticação no cron, cliente pg_dump compatível,
armazenamento temporário, recuperação e versionamento do bucket.

[Format]
Entregue script completo em um bloco bash, configuração cron ilustrativa,
pré-requisitos e plano de teste. Não alegue execução contra AWS ou banco.
```

## Modelo

**OpenAI — GPT-6, Codex nesta conversa**, escolhido pela capacidade de gerar scripts com tratamento de erro e revisar retenção. Produção assistida em 02/11/2026, contexto compartilhado; sem chamada independente de API.

## Output

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
umask 077
export PATH=/usr/local/bin:/usr/bin:/bin
export AWS_DEFAULT_REGION=us-east-1
export AWS_PAGER=""
export PGSSLMODE="${PGSSLMODE:-require}"

WORKDIR=/var/backups/ledger
LOG=/var/log/ledger-backup.log
BUCKET=hvt-ledger-backups
PREFIX=ledger/
TMPDIR_RUN=''

# Preparar diretório/log previamente, com acesso apenas do usuário de backup.
exec >>"$LOG" 2>&1
log() { printf '%s %s\n' "$(date -u +%FT%TZ)" "$*"; }
cleanup() {
  local rc=$?
  trap - EXIT
  if [[ -n "$TMPDIR_RUN" ]]; then
    rm -rf -- "$TMPDIR_RUN" || rc=1
  fi
  unset PGPASSWORD
  log "FIM exit_code=$rc"
  exit "$rc"
}
trap cleanup EXIT
trap 'rc=$?; log "ERRO linha=$LINENO exit_code=$rc"; exit "$rc"' ERR
trap 'exit 130' INT
trap 'exit 143' TERM

log 'INICIO backup Ledger'
for cmd in pg_dump gzip aws python3 flock df mktemp; do
  command -v "$cmd" >/dev/null || { log "Dependência ausente: $cmd"; exit 2; }
done
[[ -d "$WORKDIR" && -w "$WORKDIR" ]] || { log 'Diretório indisponível'; exit 2; }
[[ -n "${PGPASSWORD:-}" ]] || { log 'PGPASSWORD ausente no ambiente'; exit 2; }

exec 9>"$WORKDIR/.backup.lock"
flock -n 9 || { log 'Outra execução ativa'; exit 3; }

# Limite preventivo de 20 GiB para um dump comprimido estimado em 12 GB.
# Rever o limite conforme crescimento; não garante espaço para todo cenário.
AVAILABLE_KB=$(df -Pk "$WORKDIR" | awk 'NR==2 {print $4}')
(( AVAILABLE_KB >= 20 * 1024 * 1024 )) || { log 'Espaço livre insuficiente'; exit 4; }

TMPDIR_RUN=$(mktemp -d "$WORKDIR/.run.XXXXXXXX")
NAME="ledger_prod_$(date -u +%Y%m%dT%H%M%SZ)_$(basename "$TMPDIR_RUN").sql.gz"
FILE="$TMPDIR_RUN/$NAME"

log 'Gerando dump compactado'
pg_dump --host=ledger-db.internal.hvt.io --port=5432 \
  --username=backup_user --dbname=ledger_prod --no-password \
  --format=plain --no-owner --no-acl | gzip -6 >"$FILE.part"
gzip -t "$FILE.part"
mv -- "$FILE.part" "$FILE"

log 'Enviando backup ao S3'
aws s3 cp "$FILE" "s3://$BUCKET/$PREFIX$NAME" \
  --sse AES256 --only-show-errors

LOCAL_SIZE=$(stat -c %s "$FILE")
REMOTE_SIZE=$(aws s3api head-object --bucket "$BUCKET" \
  --key "$PREFIX$NAME" --query ContentLength --output text)
[[ "$LOCAL_SIZE" == "$REMOTE_SIZE" ]] || { log 'Tamanho remoto divergente'; exit 5; }
log "Upload confirmado: objeto=$PREFIX$NAME bytes=$LOCAL_SIZE"

# AWS CLI pagina automaticamente. Não usar --no-paginate nem --max-items.
aws s3api list-objects-v2 --bucket "$BUCKET" --prefix "$PREFIX" \
  --output json >"$TMPDIR_RUN/objects.json"

# Gera uma lista de exclusão restrita ao padrão criado por este script.
python3 - "$TMPDIR_RUN/objects.json" "$TMPDIR_RUN/delete.json" <<'PY'
import datetime as dt
import json
import re
import sys

cutoff = dt.datetime.now(dt.timezone.utc) - dt.timedelta(days=30)
with open(sys.argv[1], encoding='utf-8') as source:
    objects = json.load(source).get('Contents', [])
selected = []
for obj in objects:
    key = obj['Key']
    if not re.fullmatch(r'ledger/ledger_prod_\d{8}T\d{6}Z_\.run\.[A-Za-z0-9]+\.sql\.gz', key):
        continue
    modified = dt.datetime.fromisoformat(obj['LastModified'].replace('Z', '+00:00'))
    if modified < cutoff:
        selected.append(key)
with open(sys.argv[2], 'w', encoding='utf-8') as target:
    json.dump(selected, target)
PY

python3 - "$BUCKET" "$TMPDIR_RUN/delete.json" <<'PY'
import datetime as dt
import json
import subprocess
import sys

with open(sys.argv[2], encoding='utf-8') as source:
    keys = json.load(source)
for key in keys:
    subprocess.run(['aws', 's3api', 'delete-object', '--bucket', sys.argv[1],
                    '--key', key], check=True, stdout=subprocess.DEVNULL)
    print(f'{dt.datetime.now(dt.timezone.utc).isoformat()} Retenção: removido {key}', flush=True)
PY

log 'SUCESSO backup e retenção concluídos'
```

O script usa um único arquivo compactado, evitando um dump descompactado adicional de tamanho desconhecido. Falhas em qualquer trecho de `pg_dump | gzip` propagam pelo `pipefail`; o trap limpa arquivos locais. A validação de tamanho do objeto complementa o sucesso do upload, mas não substitui uma restauração ou uma verificação de checksum fim a fim.

Preparação ilustrativa, por administrador, para um usuário dedicado `ledgerbackup` já existente:

```bash
sudo install -d -m 700 -o ledgerbackup -g ledgerbackup /var/backups/ledger
sudo touch /var/log/ledger-backup.log
sudo chown ledgerbackup:ledgerbackup /var/log/ledger-backup.log
sudo chmod 600 /var/log/ledger-backup.log
sudo install -m 750 -o root -g ledgerbackup ledger-backup.sh /usr/local/sbin/ledger-backup.sh
```

Exemplo em `/etc/cron.d/ledger-backup`, diariamente às 02:00 **no fuso do host**:

```cron
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
0 2 * * * ledgerbackup /usr/local/sbin/ledger-backup-with-secret
```

`ledger-backup-with-secret` é uma **integração preexistente exigida**, não um arquivo criado pelo exemplo: deve obter o segredo usando a role, exportar PGPASSWORD somente para o processo e executar o script. Cron não herda variáveis exportadas em um terminal. Sem esse mecanismo, a configuração está incompleta; não colocar a senha no crontab.

Pré-requisitos: AWS CLI autenticado por instance profile, cliente `pg_dump` compatível com a versão do servidor, privilégios de leitura/backup no PostgreSQL e conectividade TLS. `require` exige criptografia, mas não valida a identidade do servidor; preferir `verify-full` com CA provisionada. A role precisa de `s3:ListBucket` limitado a `ledger/`, além de `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject` e permissões para multipart no prefixo. Segredo e KMS exigem permissões próprias quando usados.

Se o bucket usa versionamento, `delete-object` cria delete markers: a retenção acima remove a versão corrente visível, **não apaga versões antigas fisicamente**. Configurar Lifecycle para versões não correntes e multipart abandonado conforme a política de recuperação; não declarar retenção física de 30 dias sem isso. Essa política é alteração de infraestrutura separada.

Teste em ambiente isolado: credencial ausente, host inválido, disco insuficiente, upload negado, duas execuções simultâneas e objeto antigo fora do prefixo/padrão. Simular mais de uma página de objetos. Depois baixar um backup e executar `gzip -dc arquivo.sql.gz | psql ...` em banco isolado vazio, verificando tabelas e contagens. `--no-owner --no-acl` não preserva donos e grants; roles globais e configurações do servidor precisam de estratégia separada. Não houve execução contra banco ou AWS nesta sessão.

## Justificativa

**Role** delimita a experiência SRE em PostgreSQL, Bash e AWS.  
**Task** contém os parâmetros reais do exercício e critérios de erro, retenção e segurança.  
**Format** exige script, cron, pré-requisitos e testes, permitindo avaliar também as condições operacionais.

## Avaliação crítica

O controle de concorrência, o streaming gzip e o prefixo restrito resolvem riscos importantes. O exemplo depende da integração de segredo já descrita no enunciado, de permissões e de um teste de restauração. A retenção de versões antigas não deve ser confundida com a exclusão de objetos atuais; uma nova iteração poderia acrescentar configuração Lifecycle validada em IaC.
