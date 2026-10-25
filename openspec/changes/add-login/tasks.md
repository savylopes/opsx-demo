# Tasks

## 1. Fundação do projeto

- [ ] 1.1 Criar o monorepo com npm workspaces (`apps/web`, `apps/api`), `tsconfig` base, `.gitignore`, `.nvmrc` (Node 22) e `.env.example` com todas as variáveis do design. Verificar que `npm install` na raiz conclui sem erros.
- [ ] 1.2 Criar o `docker-compose.yml` com `postgres:16` (porta 5432, com volume) e `ghcr.io/navikt/mock-oauth2-server` (porta 8080). Verificar que `docker compose up -d` sobe os dois e que `curl http://localhost:8080/default/.well-known/openid-configuration` retorna JSON.
- [ ] 1.3 Esqueleto da API Fastify com `GET /api/health` e script `dev`. Verificar com `curl localhost:3000/api/health` → 200.
- [ ] 1.4 Esqueleto do front React + Vite + React Router com proxy de `/api` para `:3000`. Verificar que `curl localhost:5173/api/health` → 200 via proxy.

## 2. Banco de dados

- [ ] 2.1 Definir o schema Drizzle de `users` (UNIQUE issuer+subject) e `sessions` (id_hash PK, FK com cascade), gerar a migration com drizzle-kit e criar o script `db:migrate`. Verificar que `npm run db:migrate` cria as tabelas (`\d users` e `\d sessions` no psql).
- [ ] 2.2 Implementar o repositório de usuários (upsert por issuer+subject) e de sessões (criar com hash SHA-256, buscar válida por token, apagar), com testes Vitest contra o Postgres do compose: upsert não duplica e atualiza o perfil; sessão expirada não é retornada; o banco guarda só o hash.

## 3. Configuração e trava de produção

- [ ] 3.1 Carregar e validar a configuração de ambiente (`AUTH_PROVIDER`, `OIDC_*`, `SESSION_SECRET`, `DATABASE_URL`), falhando com mensagem clara se faltar algo. Recusar o início com `AUTH_PROVIDER=fake` + `NODE_ENV=production`. Verificar com testes Vitest dos dois cenários do requisito "Provedor fake apenas fora de produção".
- [ ] 3.2 Implementar `GET /api/auth/config` retornando `{ providerName }`, com teste Vitest para `fake` e `chatgpt`.

## 4. Fluxo OIDC na API

- [ ] 4.1 Implementar `GET /api/auth/login` com `openid-client` (discovery, PKCE S256, state, nonce, escopos `openid profile email`) e o cookie assinado `auth_tx` (10 min). Verificar com testes Vitest: redirect contém os parâmetros exigidos, e duas tentativas geram valores diferentes.
- [ ] 4.2 Implementar `GET /api/auth/callback`: validar `auth_tx`, trocar o código, validar o ID token, fazer o upsert do usuário, criar a sessão, definir o cookie `sid` (HttpOnly, SameSite=Lax, Path=/, Secure em produção, 7 dias), apagar `auth_tx` e redirecionar para `/`. Em erro, redirecionar para `/login?error=...` sem sessão. Verificar com testes Vitest dos cenários de sucesso (via fake OIDC), state divergente, `error` do provedor e ausência de tentativa.
- [ ] 4.3 Implementar o middleware de sessão e `GET /api/me` (200 com `id`, `name`, `email`, `avatarUrl`; 401 sem sessão ou com sessão inválida/expirada). Verificar com testes Vitest.
- [ ] 4.4 Implementar `POST /api/auth/logout` (apaga a sessão e limpa o cookie). Verificar com teste Vitest: depois do logout, o mesmo token recebe 401 em `/api/me`.

## 5. Frontend

- [ ] 5.1 Implementar o `AuthProvider` (carrega `/api/me`) e as rotas protegidas: `/` sem sessão vai para `/login`, e `/login` com sessão vai para `/`. Verificar manualmente no navegador e no e2e da tarefa 6.1.
- [ ] 5.2 Página `/login` com o botão "Entrar com <providerName>" (link para `/api/auth/login`) e mensagem de erro quando houver `?error=`. Verificar com teste de componente (Vitest + Testing Library) do rótulo e da mensagem de erro.
- [ ] 5.3 Página `/` com "Olá, <nome>" e botão "Sair" (POST logout, depois navega para `/login`). Verificar com teste de componente.

## 6. Integração e documentação

- [ ] 6.1 Teste e2e Playwright: `/` → `/login` → login como `alice` no fake → `/` mostra "Olá, alice" → "Sair" → `/login`; e um segundo caso em que `bob` vê o próprio nome. Verificar que `npm run test:e2e` passa com o compose ativo.
- [ ] 6.2 README com: pré-requisitos, `docker compose up -d`, `npm run db:migrate`, `npm run dev`, usuários de teste, variáveis de ambiente e como trocar para ChatGPT (`AUTH_PROVIDER=chatgpt`, issuer, client ID, callback registrado). Verificar seguindo o README a partir de um clone limpo.

## Workflow follow-up

- Quando houver client ID do SIWC, fazer um smoke test manual do login com ChatGPT.
- Próxima change: CRUD de tarefas por usuário (com teste de isolamento alice/bob).
- Arquivar a change após revisão.
