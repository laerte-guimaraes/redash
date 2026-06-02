# Redash

Instância do [Redash](https://redash.io), containerizada com Docker e servida via nginx com SSL.

## Arquitetura

```
Internet → Cloudflare (proxy) → nginx (SSL termination) → Redash (porta 5000)
                                                         → PostgreSQL
                                                         → Redis
```

**Serviços Docker:**

| Serviço            | Função                                      |
|--------------------|---------------------------------------------|
| `server`           | API e frontend Redash (porta 5000, loopback) |
| `scheduler`        | Agendamento de queries                      |
| `scheduled_worker` | Execução de queries agendadas               |
| `adhoc_worker`     | Execução de queries manuais                 |
| `worker`           | Filas de emails, notificações e tarefas     |
| `redis`            | Fila de jobs                                |
| `postgres`         | Banco de dados da aplicação                 |
| `nginx`            | Proxy reverso com SSL (portas 80 e 443)     |

## Pré-requisitos

- Ubuntu 22.04+ (ou similar)
- Acesso root ao servidor
- Domínio apontando para o servidor via Cloudflare (proxy ativado)
- Cloudflare Origin Certificate para o domínio

## Primeiro deploy

### 1. Clonar o repositório

```bash
git clone <repo-url> /opt/redash
cd /opt/redash
```

### 2. Adicionar os certificados SSL

Gere um **Cloudflare Origin Certificate** no painel da Cloudflare:
> SSL/TLS → Origin Server → Create Certificate → escolha o domínio (ex: `YOUR_HOST`)

```bash
mkdir -p /opt/redash/nginx/certs
nano /opt/redash/nginx/certs/origin.crt   # cole o certificado
nano /opt/redash/nginx/certs/origin.key   # cole a chave privada
```

### 3. Rodar o setup

```bash
sudo bash setup.sh
```

O script:
- Instala o Docker se necessário
- Gera o `.env` com secrets aleatórios a partir do `.env.example`
- Inicializa o banco de dados
- Sobe todos os serviços

### 4. (Opcional) Configurar Google OAuth

Edite o `.env` e preencha:

```env
REDASH_GOOGLE_CLIENT_ID=seu-client-id
REDASH_GOOGLE_CLIENT_SECRET=seu-client-secret
```

URI de redirecionamento autorizado no Google Console:
```
https://{YOUR_HOST}/oauth/google
```

Depois reinicie:
```bash
docker compose up -d
```

## Variáveis de ambiente

Copie `.env.example` para `.env` e ajuste os valores. O arquivo `.env` **nunca deve ser commitado**.

| Variável                    | Descrição                                  |
|-----------------------------|--------------------------------------------|
| `REDASH_COOKIE_SECRET`      | Secret para assinar cookies (32 hex chars) |
| `REDASH_SECRET_KEY`         | Secret key da aplicação (32 hex chars)     |
| `REDASH_REDIS_URL`          | URL do Redis                               |
| `REDASH_DATABASE_URL`       | URL do PostgreSQL                          |
| `POSTGRES_PASSWORD`         | Senha do PostgreSQL                        |
| `REDASH_GOOGLE_CLIENT_ID`   | OAuth Google (opcional)                    |
| `REDASH_GOOGLE_CLIENT_SECRET` | OAuth Google (opcional)                  |

## Comandos úteis

```bash
# Status dos containers
docker compose ps

# Logs em tempo real
docker compose logs -f

# Logs de um serviço específico
docker compose logs nginx --tail=50
docker compose logs server --tail=50

# Reiniciar um serviço
docker compose restart nginx

# Parar tudo
docker compose down

# Parar e remover volumes (DESTRUTIVO — apaga o banco)
docker compose down -v
```

## Estrutura de arquivos

```
/opt/redash/
├── docker-compose.yml       # definição dos serviços
├── setup.sh                 # script de primeiro deploy
├── .env                     # secrets (não commitado)
├── .env.example             # template do .env
└── nginx/
    ├── nginx.conf           # configuração do nginx
    └── certs/
        ├── origin.crt       # Cloudflare Origin Certificate
        └── origin.key       # chave privada do certificado
```

## Troubleshooting

### Erro 521 (Cloudflare)

Cloudflare não consegue conectar ao servidor. Causas comuns:

1. **nginx não está rodando:** `docker compose ps` → reiniciar com `docker compose restart nginx`
2. **Certificados ausentes:** confirme que `nginx/certs/origin.crt` e `origin.key` existem
3. **Firewall:** verifique se as portas 80 e 443 estão abertas no Cloud Firewall do Digital Ocean (painel → Networking → Firewalls)

### SSL não funciona (SSL_ERROR_SYSCALL)

O nginx está aceitando conexão TCP mas não responde SSL. Verifique:

```bash
# Config carregada pelo nginx (deve ter bloco "listen 443 ssl")
docker compose exec nginx nginx -T

# Certificado válido
openssl x509 -in nginx/certs/origin.crt -noout -subject

# Chave e certificado batem
openssl x509 -noout -modulus -in nginx/certs/origin.crt | md5sum
openssl rsa -noout -modulus -in nginx/certs/origin.key | md5sum
```

Se o `nginx -T` não mostrar o bloco 443, o `nginx.conf` no servidor está desatualizado — atualize e reinicie.

### Serviços não sobem (`.env` não encontrado)

```bash
# Gere o .env a partir do exemplo
cp .env.example .env
# Edite com secrets reais e depois suba
docker compose up -d
```

## Notas para IA

- **Stack:** Redash (Python/Flask) + PostgreSQL + Redis + nginx + Docker Compose
- **Deploy:** Digital Ocean droplet, domínio `YOUR_HOST` via Cloudflare com proxy ativo
- **SSL:** Cloudflare Origin Certificate (não Let's Encrypt). O modo SSL no Cloudflare deve ser **Full** ou **Full (strict)**
- **Secrets:** o arquivo `.env` nunca está no repositório. Para recriar, use `setup.sh` ou copie `.env.example` manualmente
- **nginx:** o arquivo `nginx/nginx.conf` é montado como read-only no container. Após qualquer edição, é necessário `docker compose restart nginx`
- **Porta 5000:** o Redash escuta apenas em `127.0.0.1:5000` no host — acesso externo é exclusivamente via nginx
- **Banco de dados:** o volume `postgres_data` persiste os dados. Nunca rode `docker compose down -v` em produção sem backup
- **Inicialização do banco:** só precisa rodar uma vez — `docker compose run --rm server create_db`
- **Problema histórico:** em Jun/2026, o nginx foi deployado sem o bloco HTTPS (porta 443) no `nginx.conf` do servidor, causando erro 521. Solução: sobrescrever o arquivo com a versão correta e reiniciar o nginx
