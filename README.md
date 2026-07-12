# StreamTube — Plataforma de Compartilhamento de Vídeos

Projeto da disciplina **Desenvolvimento de Aplicações de IA** do MBA de Engenharia de Software com IA da [Full Cycle](https://fullcycle.com.br).

Este é um projeto greenfield desenvolvido para demonstrar como construir uma aplicação do zero utilizando IA de forma adequada no processo de desenvolvimento.

## Professor

<a href="https://github.com/argentinaluiz">
    <img src="https://avatars.githubusercontent.com/u/4926329?v=4?s=100" width="100px;" alt=""/>
    <br />
    <sub>
        <b>Luiz Carlos</b>
    </sub>
</a>

---

## Quadro Branco

- [Quadro Branco](./whiteboard.png)

---

## 🎨 Design System (Figma)

- [FC Tube.fig](./FC%20Tube.fig) — arquivo-fonte do **design system** do projeto no Figma.

Contém os fundamentos visuais do StreamTube — tokens (cores, tipografia, espaçamento, raios), componentes e as telas da plataforma. É a referência de design para a implementação do frontend: os componentes em `next-frontend/components/ui` (shadcn) e os tokens em `next-frontend/app/globals.css` derivam deste arquivo. Abra-o no Figma (`Arquivo → Importar`) para consultar especificações e estados visuais.

---

## 📋 Pré-requisitos

- Docker e Docker Compose
- Node.js v25+ (para rodar os testes E2E do Playwright no host)
- npm

## 🏗️ Arquitetura

O projeto é um monorepo baseado em containers Docker. Cada subprojeto sobe sua própria stack via `docker compose`.

- **Frontend** (Next.js 16, App Router + React Server Components) — interface da plataforma. Segue o **modelo BFF**: o navegador nunca chama a API NestJS diretamente; todo tráfego passa por Route Handlers same-origin em `app/api/**`, que fazem proxy server-side para a API.
- **API** (NestJS 11) — regras de negócio, autenticação (JWT + refresh token rotation), envio de e-mails e acesso ao banco.
- **Database** (PostgreSQL 17) — usuários, canais, tokens de autenticação e metadados de vídeos.
- **Email Service** (Mailpit) — captura os e-mails transacionais (confirmação de conta e recuperação de senha) em uma UI local.
- **Video Worker** (FFmpeg) — consumidor BullMQ standalone que processa os vídeos enviados (thumbnail, duração, metadados).
- **Object Storage** (S3/MinIO) — arquivos de vídeo originais e thumbnails.
- **Message Queue** (BullMQ + Redis) — fila de jobs de processamento de vídeo.

O diagrama de arquitetura completo (C4) está em `docs/diagrams/software-arch.mermaid`.

## 🚀 Como rodar

Os dois subprojetos têm stacks Docker **separadas**. Suba primeiro o backend, rode as migrations e depois o frontend.

### 1. Backend (NestJS + PostgreSQL + Mailpit + Redis + MinIO + Video Worker)

```bash
cd nestjs-project

# Sobe API, banco, Mailpit, Redis, MinIO e o video-worker
docker compose up -d

# Instala dependências (apenas na primeira vez)
docker compose exec nestjs-api npm install

# Cria o schema do banco (obrigatório — synchronize está desabilitado)
docker compose exec nestjs-api npm run migration:run

# Sobe o servidor de desenvolvimento em watch mode
docker compose exec -d nestjs-api npm run start:dev
```

Serviços disponíveis:

| Serviço | URL / Porta |
|---------|-------------|
| API NestJS | http://localhost:3000 |
| PostgreSQL | `localhost:5432` (db/user/senha: `streamtube`) |
| Mailpit (UI de e-mails) | http://localhost:8025 |
| Redis (fila BullMQ) | `localhost:6379` |
| MinIO (API S3) | http://localhost:9000 |
| MinIO (console web) | http://localhost:9001 (usuário/senha: `streamtube`) |
| Swagger | http://localhost:3000/api/docs — habilite com `SWAGGER_ENABLED=true` |

O bucket do MinIO é criado automaticamente pela própria API no boot — nenhuma ação manual necessária.

#### ⚠️ Configuração necessária: resolver o hostname `minio` no host

A API gera **URLs pré-assinadas de upload** (`PUT` direto no MinIO) apontando para `http://minio:9000` — esse é o nome do serviço Docker, que só é resolvido *dentro* da rede do Compose. Para completar o upload de um vídeo a partir do seu navegador ou de um `curl` rodando no Windows (fora do container), esse hostname precisa resolver para `127.0.0.1`.

Adicione a seguinte linha ao arquivo hosts do Windows (`C:\Windows\System32\drivers\etc\hosts`) — requer um editor aberto **como Administrador** (Notepad ou PowerShell):

```
127.0.0.1 minio
```

Via PowerShell como Administrador:

```powershell
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "127.0.0.1 minio"
```

Sem esse ajuste, o `POST /videos` funciona normalmente (cria o registro e retorna a URL pré-assinada), mas o `PUT` do arquivo na URL retornada falha por não resolver o hostname `minio`.

### 2. Frontend (Next.js)

```bash
cd next-frontend

# Garanta que o .env.local existe (veja .env.example)
# API_URL aponta para o backend; SESSION_PASSWORD protege a sessão (iron-session)

docker compose up -d
docker compose exec next-frontend npm install        # apenas na primeira vez
docker compose exec -d next-frontend npm run dev
```

A aplicação ficará disponível em **http://localhost:3001**.

> As stacks são separadas, então o frontend acessa o backend via `host.docker.internal:3000` (configurado em `next-frontend/.env.local` e no `extra_hosts` do compose).

## 🧪 Testes

### Backend (Jest)

```bash
cd nestjs-project
docker compose exec nestjs-api npm test               # unitários + integração
docker compose exec nestjs-api npm run test:e2e       # end-to-end (HTTP via supertest)
docker compose exec nestjs-api npm run test:cov       # cobertura
```

Sufixos: `*.spec.ts` (unitário), `*.integration-spec.ts` (integração com banco real), `*.e2e-spec.ts` (end-to-end). Testes de integração/e2e rodam com `--runInBand`.

### Frontend (Vitest + Playwright)

```bash
cd next-frontend
docker compose exec next-frontend npm test            # unitários + integração (Vitest + MSW)
npx playwright test                                   # end-to-end (no host, com dev server em MSW_ENABLED=true)
```

Sufixos: `*.test.ts(x)` (unitário), `*.integration.test.ts(x)` (Route Handlers com MSW), `*.e2e-spec.ts` (Playwright). MSW intercepta as chamadas à API NestJS — os testes nunca batem no backend real.

## ✅ Funcionalidades implementadas

**Fase 01 — Configuração base** e **Fase 02 — Autenticação** estão concluídas (backend + frontend).

### Autenticação (Fase 02)

Fluxo completo de **cadastro → confirmação por e-mail → login → recuperação de senha**, com canal criado automaticamente para cada usuário (a partir do prefixo do e-mail).

Endpoints da API (`nestjs-project`):

| Método & Rota | Descrição |
|---------------|-----------|
| `POST /auth/register` | Cadastro de usuário (cria usuário + canal) |
| `GET /auth/confirm-email?token=` | Confirmação de conta via link do e-mail |
| `POST /auth/resend-confirmation` | Reenvio do e-mail de confirmação |
| `POST /auth/login` | Login (retorna access + refresh token) |
| `POST /auth/refresh` | Rotação de refresh token (com family + grace period) |
| `POST /auth/logout` | Revoga os refresh tokens da sessão |
| `POST /auth/forgot-password` | Solicita e-mail de recuperação de senha |
| `POST /auth/reset-password` | Redefine a senha via token |
| `GET /auth/me` | Dados do usuário autenticado (protegido por JWT) |

Telas e Route Handlers BFF (`next-frontend`):

- `/(auth)/signup`, `/(auth)/login`, `/(auth)/forgot-password` — formulários com React Hook Form + Zod e validação inline.
- `app/api/auth/{signup,login,logout,forgot-password}` — proxy same-origin para a API.

Segurança: senhas com **Argon2**, **JWT** com `JwtAuthGuard` global (opt-out via `@Public()`), **rotação de refresh token** com detecção de reuso, **rate limiting** (`ThrottlerGuard`) nos endpoints de auth, e sessão no navegador via **iron-session** (cookies HTTP-only).

### Upload e processamento de vídeos (Fase 03 — backend)

Pipeline completo de upload direto ao Object Storage (padrão pre-signed URL) e processamento assíncrono via fila. **Ainda não há tela no frontend para este fluxo** — teste via Swagger UI (`http://localhost:3000/api/docs`) seguindo o passo a passo abaixo.

Endpoints da API (`nestjs-project`):

| Método & Rota | Auth | Descrição |
|---------------|------|-----------|
| `POST /videos` | JWT | Cria um vídeo em rascunho e retorna a URL pré-assinada de upload |
| `POST /videos/:id/upload-complete` | JWT (dono) | Confirma o upload e enfileira o processamento (FFmpeg) |
| `GET /videos/:id` | Público | Consulta metadados e status do vídeo |
| `GET /videos/:id/stream` | Público | Stream com suporte a range requests (`206`) |
| `GET /videos/:id/download` | JWT | Download do arquivo completo (`Content-Disposition: attachment`) |
| `GET /channels/:channelId/videos` | JWT (dono) | Lista vídeos do canal (paginado) |
| `PATCH /videos/:id` | JWT (dono) | Atualiza o título |
| `DELETE /videos/:id` | JWT (dono) | Remove o vídeo (registro + arquivos no storage) |

Ciclo de vida do vídeo: `draft` → `processing` → `ready` (ou `error`, se o processamento falhar).

#### Passo a passo — testando o upload pelo Swagger

Pré-requisito: a entrada `127.0.0.1 minio` já aplicada no arquivo hosts (seção acima).

1. **Login e autorização** — `POST /auth/login` com um usuário já confirmado → copie o `access_token` da resposta → clique em **Authorize** (cadeado no topo do Swagger UI), cole o token (sem o prefixo `Bearer`) e confirme.

2. **Iniciar o upload** — `POST /videos` → "Try it out" → envie:
   ```json
   {
     "title": "Meu primeiro vídeo",
     "file_size": 1048576,
     "mime_type": "video/mp4"
   }
   ```
   `file_size` deve ser o tamanho real (em bytes) do arquivo a enviar. A resposta traz `id` (uuid do vídeo), `status: "draft"` e `upload_url` (URL pré-assinada).

3. **Enviar o arquivo** — o Swagger não tem campo para `PUT` bruto; use `curl` no terminal (Windows, fora do container):
   ```bash
   curl -X PUT "URL_COPIADA_DO_PASSO_2" \
     -H "Content-Type: video/mp4" \
     --upload-file "caminho/do/seu/video.mp4"
   ```
   Deve responder `200 OK` sem corpo.

4. **Confirmar o upload** — `POST /videos/{id}/upload-complete` com o `id` do passo 2 → muda o status para `processing` e enfileira o job no `video-worker`.

5. **Acompanhar o processamento** — `GET /videos/{id}` (público) repetidamente até `status` virar `ready` (ou `error`). Para ver o FFmpeg rodando em tempo real:
   ```bash
   cd nestjs-project
   docker compose logs video-worker -f
   ```

6. **Baixar o vídeo** — com o `access_token` autorizado no Swagger, abra `GET /videos/{id}/download` → "Try it out" → Execute → use o link **"Download file"** na resposta. Alternativa via `curl`:
   ```bash
   curl -H "Authorization: Bearer SEU_ACCESS_TOKEN" \
     "http://localhost:3000/videos/SEU_VIDEO_ID/download" \
     -o video-baixado.mp4
   ```
   Retorna `409` se o vídeo ainda não estiver `ready`.

## 🛠️ Estrutura do Projeto

```
green-field-ia-project/
├── docs/
│   ├── project-plan.md                  # Planejamento geral do projeto
│   ├── phases/                          # Planos e implementação por fase
│   │   ├── phase-01-configuracao-base/
│   │   ├── phase-02-auth/               # Auth (backend)
│   │   └── phase-02-auth-frontend/      # Auth (frontend)
│   └── diagrams/
│       └── software-arch.mermaid        # Diagrama de arquitetura (C4)
├── nestjs-project/                      # Backend API (NestJS 11)
│   ├── src/
│   │   ├── auth/                        # Cadastro, login, JWT, refresh, reset de senha
│   │   ├── users/                       # Entidade e serviço de usuários
│   │   ├── channels/                    # Canal 1:1 por usuário (nickname do e-mail)
│   │   ├── videos/                      # Upload, metadados e endpoints REST de vídeo
│   │   ├── video-processing/            # Processor BullMQ (FFmpeg) — só no worker standalone
│   │   ├── storage/                     # Cliente S3/MinIO, URLs pré-assinadas
│   │   ├── mail/                        # Envio de e-mails (templates Handlebars)
│   │   ├── common/                      # Filtros, pipes e exceptions de domínio
│   │   ├── config/                      # Configs namespaced (Joi)
│   │   └── database/                    # data-source, migrations e seeds
│   ├── test/                            # Testes e2e
│   ├── compose.yaml                     # Docker Compose (API + PostgreSQL + Mailpit + Redis + MinIO + worker)
│   ├── Dockerfile.dev
│   └── Dockerfile.worker                # Imagem do video-worker (BullMQ standalone)
├── next-frontend/                       # Frontend (Next.js 16, App Router)
│   ├── app/                             # Rotas, layouts, páginas e Route Handlers BFF
│   ├── components/                      # Componentes de auth, UI (shadcn) e ícones
│   ├── lib/                             # env, api (openapi-fetch), auth/session
│   ├── mocks/                           # MSW (handlers + server)
│   ├── tests/                           # E2E (Playwright)
│   ├── compose.yaml                     # Docker Compose (dev server)
│   └── Dockerfile.dev
├── CLAUDE.md                            # Instruções para IA
├── FC Tube.fig                          # Design system do projeto (Figma)
├── whiteboard.png                       # Quadro branco do projeto
└── README.md
```

## 📚 Fases do Projeto

| Fase | Descrição | Status |
|------|-----------|--------|
| **01** | Configuração Base do Projeto | ✅ Concluída |
| **02** | Cadastro, Login e Gerenciamento de Conta | ✅ Concluída |
| **03** | Upload e Processamento de Vídeos | ✅ Concluída (backend) — frontend pendente |
| **04** | Gerenciamento de Vídeos e Canal | ⏳ Planejada |
| **05** | Página de Visualização do Vídeo | ⏳ Planejada |
| **06** | Interações Sociais (Likes, Comentários, Inscrições) | ⏳ Planejada |
| **07** | Página Inicial, Busca e Finalização | ⏳ Planejada |

Detalhes completos em `docs/project-plan.md`.

## 📖 Stack Tecnológica

| Camada | Tecnologia |
|--------|------------|
| Frontend | Next.js 16, React 19, TypeScript, Tailwind CSS 4, shadcn/ui, React Hook Form + Zod, iron-session, openapi-fetch |
| Backend | NestJS 11, TypeScript, TypeORM, JWT, Argon2, Mailer (Handlebars) |
| Vídeo | BullMQ + Redis (fila), FFmpeg (processamento), AWS SDK S3 (client MinIO) |
| Banco de Dados | PostgreSQL 17 |
| Object Storage (dev) | MinIO |
| E-mail (dev) | Mailpit |
| Containerização | Docker, Docker Compose |
| Testes | Jest, Supertest (backend); Vitest, MSW, Playwright (frontend) |
| Qualidade | ESLint, Prettier |
</content>
