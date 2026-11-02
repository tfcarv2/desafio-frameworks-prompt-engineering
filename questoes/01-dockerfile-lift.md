# Questão 01 — Dockerfile para o Lift

## Prompt

```text
[Role]
Atue como engenheiro de plataforma especialista em containers Python e Kubernetes.

[Task]
Produza um Dockerfile de produção para o Lift, uma API Flask na porta 8080.
Estrutura: app.py, requirements.txt, lib/auth.py, lib/storage.py e tests/test_app.py.
requirements.txt contém exatamente:
Flask==3.0.0
gunicorn==21.2.0
requests==2.31.0
python-dotenv==1.0.0
psycopg2-binary==2.9.9
O comando obrigatório é gunicorn --bind 0.0.0.0:8080 --workers 4 app:app.
DATABASE_URL e API_KEY serão fornecidas em runtime pelo Kubernetes: não grave
valores nem arquivos .env na imagem. Preserve as dependências do enunciado.
Use imagem Python slim com versão explícita, instalação sem cache, cópia de
requirements antes do código para aproveitar cache de build, usuário não-root,
logs para stdout/stderr e CMD em exec form. Copie apenas os arquivos de runtime.
Inclua .dockerignore. Explique as premissas sobre versão Python, gravabilidade
de /tmp e endpoints de health check, sem inventar endpoint existente. Não diga
que o build foi testado. Cite o que falta validar antes de produção.

[Format]
Responda com: 1) Dockerfile em bloco dockerfile; 2) .dockerignore em bloco text;
3) explicação breve das decisões; 4) comandos de build e smoke test que usem
variáveis locais já exportadas, sem mostrar credenciais.
```

## Modelo

**OpenAI — GPT-6, Codex nesta conversa**, escolhido pela geração e revisão de código de infraestrutura. Resposta produzida nesta sessão assistida em 02/11/2026, com contexto compartilhado; sem chamada independente de API.

## Output

### Dockerfile

```dockerfile
FROM python:3.11-slim-bookworm

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

RUN groupadd --gid 10001 app \
    && useradd --uid 10001 --gid app --no-create-home \
       --shell /usr/sbin/nologin app

COPY requirements.txt ./
RUN python -m pip install --no-cache-dir -r requirements.txt

COPY app.py ./
COPY lib/ ./lib/

USER 10001:10001
EXPOSE 8080

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "--access-logfile", "-", "--error-logfile", "-", "app:app"]
```

### .dockerignore

```text
.git
.venv
venv
__pycache__
*.py[cod]
.pytest_cache
.coverage
htmlcov
tests
.env
.env.*
*.pem
*.key
```

Assumo compatibilidade da aplicação com Python 3.11. A imagem tem versão minor e distribuição explícitas; para reprodução byte a byte, resolver e fixar o digest aprovado no pipeline, mantendo política de atualização. Não invento um digest. Como há wheels de psycopg2-binary para plataformas usuais, não instalo compiladores; outra arquitetura precisa de build validado.

O código fica sob propriedade de root, legível pelo usuário da aplicação. `/tmp` precisa estar gravável para o Gunicorn; ao usar root filesystem somente leitura no Kubernetes, montar um `emptyDir` limitado em `/tmp`. As variáveis obrigatórias devem ser validadas pela aplicação ao iniciar. Não há `HEALTHCHECK` HTTP com caminho inventado; configure probes após confirmar os endpoints da API.

```bash
docker build --pull -t lift:1.0.0 ./lift
# DATABASE_URL e API_KEY devem estar exportadas no terminal local.
docker run --rm --name lift-smoke -p 8080:8080 \
  -e DATABASE_URL -e API_KEY lift:1.0.0
# Em outro terminal, verificar uma rota conhecida e os logs:
docker logs lift-smoke
```

Validar startup, rotas, acesso ao PostgreSQL, shutdown por SIGTERM, consumo de memória com quatro workers e scan de vulnerabilidades. As versões de dependência do exercício foram preservadas; isso não equivale a considerá-las atualizadas ou sem vulnerabilidades.

## Justificativa

**Role** define a especialidade em plataforma, Python e Kubernetes.  
**Task** informa estrutura, dependências, comando e restrições de segurança.  
**Format** pede quatro partes explícitas, tornando a saída diretamente verificável.

## Avaliação crítica

O resultado atende ao comando e evita segredos e execução como root. Não usa multi-stage sem necessidade: não há etapa de compilação especificada. Permanecem pendentes o build com o código real, a fixação do digest e a confirmação dos endpoints; nenhum desses testes foi declarado como realizado.
