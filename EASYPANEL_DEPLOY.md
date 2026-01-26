# Deploy no Easypanel

Este guia explica como fazer deploy da aplicação no Easypanel usando Docker Compose e imagens do GitHub Container Registry.

## Pré-requisitos

1. Conta no Easypanel
2. Repositório GitHub com GitHub Actions configurado para build de imagens Docker
3. Imagem Docker publicada no GitHub Container Registry (ghcr.io)

## Passo 1: Configurar GitHub Actions para Build da Imagem

Crie o arquivo `.github/workflows/docker-build.yml`:

```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ main, master ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main, master ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            BUILD_COMMIT_SHA=${{ github.sha }}
```

## Passo 2: Configurar o Easypanel

### 2.1 Criar Novo Projeto

1. Acesse seu painel do Easypanel
2. Clique em "Create Project"
3. Escolha "Docker Compose"

### 2.2 Configurar Docker Compose

1. Cole o conteúdo do arquivo `docker-compose.easypanel.yml`
2. **IMPORTANTE**: Substitua `ghcr.io/seu-usuario/seu-repo:latest` pelo caminho correto da sua imagem
   - Exemplo: `ghcr.io/johndoe/my-finance-app:latest`

### 2.3 Configurar Variáveis de Ambiente

Configure as seguintes variáveis obrigatórias no Easypanel:

```bash
# Obrigatórias
SECRET_KEY_BASE=<gere com: openssl rand -hex 64>
POSTGRES_PASSWORD=<senha segura do postgres>

# Recomendadas
POSTGRES_USER=sure_user
POSTGRES_DB=sure_production
RAILS_FORCE_SSL=true
RAILS_ASSUME_SSL=true
ONBOARDING_STATE=invite_only

# Opcionais (AI features - incorre custos!)
OPENAI_ACCESS_TOKEN=sk-...
OPENAI_MODEL=gpt-4
```

### 2.4 Expor o Serviço Web

1. No Easypanel, selecione o serviço `web`
2. Configure a porta interna: `3000`
3. O Easypanel irá gerar automaticamente uma URL pública

### 2.5 Configurar Domínio (Opcional)

1. No Easypanel, vá em "Domains"
2. Adicione seu domínio personalizado
3. Configure SSL (Let's Encrypt automático)

## Passo 3: Deploy

1. Clique em "Deploy" no Easypanel
2. Aguarde o pull da imagem e inicialização dos serviços
3. Configure o serviço `web` para ser exposto na porta `3000`
4. O Easypanel irá gerar uma URL pública automaticamente
5. Verifique os logs para confirmar que tudo está funcionando

## Estrutura dos Serviços

O docker-compose cria 4 serviços:

- **web**: Servidor Rails (porta interna 3000, exposta pelo Easypanel)
- **worker**: Sidekiq para jobs em background (não exposto)
- **db**: PostgreSQL 16 (interno)
- **redis**: Redis para cache e Sidekiq (interno)

**Nota**: O Easypanel gerencia automaticamente a exposição de portas. Você só precisa configurar qual serviço expor (web) e qual porta interna ele usa (3000).

## Volumes Persistentes

- `app-storage`: Arquivos de upload da aplicação
- `postgres-data`: Dados do banco PostgreSQL
- `redis-data`: Dados do Redis

## Healthchecks

Todos os serviços têm healthchecks configurados:
- Web: verifica endpoint `/up`
- Worker: verifica status do Sidekiq
- DB: verifica conexão PostgreSQL
- Redis: verifica comando PING

## Atualizações

Para atualizar a aplicação:

1. Faça push do código para o GitHub
2. GitHub Actions fará build da nova imagem automaticamente
3. No Easypanel, clique em "Redeploy" ou configure auto-deploy

## Troubleshooting

### Imagem não encontrada
- Verifique se o GitHub Actions rodou com sucesso
- Confirme que a imagem está pública ou que o Easypanel tem acesso
- Verifique o nome da imagem no docker-compose

### Erro de conexão com banco
- Verifique se `POSTGRES_PASSWORD` está configurado
- Confirme que o serviço `db` está healthy
- Verifique os logs: `docker-compose logs db`

### Worker não processa jobs
- Verifique logs: `docker-compose logs worker`
- Confirme que Redis está rodando
- Verifique variáveis de ambiente

### Erro 500 na aplicação
- Verifique se `SECRET_KEY_BASE` está configurado
- Confirme que as migrations rodaram
- Verifique logs: `docker-compose logs web`

## Comandos Úteis

```bash
# Ver logs de todos os serviços
docker-compose logs -f

# Ver logs de um serviço específico
docker-compose logs -f web

# Rodar migrations manualmente
docker-compose exec web bundle exec rails db:migrate

# Abrir console Rails
docker-compose exec web bundle exec rails console

# Verificar status dos serviços
docker-compose ps

# Reiniciar um serviço
docker-compose restart web
```

## Segurança

- ✅ Sempre use senhas fortes para `POSTGRES_PASSWORD`
- ✅ Gere um `SECRET_KEY_BASE` único e seguro
- ✅ Configure `RAILS_FORCE_SSL=true` em produção
- ✅ Use `ONBOARDING_STATE=invite_only` para controlar acesso
- ✅ Mantenha as imagens atualizadas
- ✅ Configure backups regulares do volume `postgres-data`

## Backup

Para fazer backup dos dados:

```bash
# Backup do banco
docker-compose exec db pg_dump -U sure_user sure_production > backup.sql

# Backup dos arquivos
docker-compose exec web tar czf /tmp/storage-backup.tar.gz /rails/storage
docker-compose cp web:/tmp/storage-backup.tar.gz ./storage-backup.tar.gz
```

## Suporte

Para mais informações:
- [Documentação do Easypanel](https://easypanel.io/docs)
- [GitHub Container Registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
