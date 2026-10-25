# Design

## Context

O repositório está vazio, com apenas `openspec/`. Toda a estrutura nasce nesta change. Motivação e escopo estão em proposal.md, e o comportamento esperado em `specs/user-auth/spec.md`.

Restrições:
- Ainda não há client ID do Sign in with ChatGPT (SIWC). O fluxo precisa funcionar e ser testável de ponta a ponta só com um provedor OIDC fake local.
- O SIWC é OIDC padrão (Authorization Code + PKCE, escopos `openid profile email`, client ID `oaiapp_...`, callback registrado por ambiente). Portanto, um cliente OIDC genérico atende os dois provedores.

## Goals / Non-Goals

**Goals:**
- Um cliente OIDC só no backend, com o provedor escolhido apenas por variáveis de ambiente.
- Ambiente local reproduzível: `docker compose up -d` (infra) + `npm run dev` (app).
- Login testável automaticamente (API e e2e), com dois usuários fake distintos para os testes de isolamento da próxima change.

**Non-Goals:**
- Validar o login real com ChatGPT, que depende do client ID.
- Usar o access token do provedor (plan usage / Responses API). Usamos só o ID token para identidade.
- Deploy em produção, refresh tokens, "lembrar de mim" e multiplos provedores ao mesmo tempo.
- O fluxo de logout no provedor (RP-initiated logout). O logout encerra só a sessão local.

## Decisions

### 1. Monorepo TypeScript com npm workspaces
```
apps/
  web/   React + Vite + TypeScript (React Router)
  api/   Node 22 + Fastify + TypeScript
docker-compose.yml   postgres + fake OIDC
```
Uma linguagem só no projeto, ferramentas comuns e fáceis de testar. Alternativas: Next.js full-stack (front e API acoplados, e o OIDC no edge/runtime complica) e Python/FastAPI (duas linguagens no projeto).

### 2. Mesma origem via proxy do Vite
Em dev, o Vite (`:5173`) faz proxy de `/api` para a API (`:3000`). Assim o navegador vê uma origem só: os cookies `SameSite=Lax` funcionam sem CORS, e a redirect URI registrada é `http://localhost:5173/api/auth/callback`. Alternativa descartada: API em outra origem com CORS + credentials, que tem mais configuração e mais superfície de erro.

### 3. Cliente OIDC: `openid-client` (v6)
É uma biblioteca certificada pela OpenID Foundation. Faz discovery, PKCE, valida o ID token (assinatura via JWKS, `iss`, `aud`, `exp`, `nonce`) e o `state`. Não escrevemos criptografia nem validação de JWT na mão.

Configuração por ambiente:
```
AUTH_PROVIDER=fake | chatgpt     # define o rótulo do botão e a trava de produção
OIDC_ISSUER=...                  # URL de discovery
OIDC_CLIENT_ID=...
OIDC_CLIENT_SECRET=              # opcional (o SIWC open-source não usa secret)
OIDC_REDIRECT_URI=http://localhost:5173/api/auth/callback
SESSION_SECRET=...               # assina o cookie da transação de login
DATABASE_URL=postgres://...
```
`GET /api/auth/config` devolve `{ providerName }` para a página de login mostrar "Entrar com Fake OIDC" ou "Entrar com ChatGPT".

### 4. Estado da transação de login em cookie assinado
`/api/auth/login` gera `state`, `nonce` e `code_verifier` e grava os três num cookie `auth_tx` assinado (`@fastify/cookie`), com `HttpOnly`, `SameSite=Lax`, `Max-Age` de 10 min e `Path=/api/auth`. O callback lê, valida e apaga esse cookie. Alternativa descartada: guardar no banco, o que exige limpeza e não traz ganho para esse tempo de vida.

`SameSite=Lax` é suficiente porque o retorno do provedor é uma navegação GET top-level.

### 5. Sessões no servidor, não JWT
O cookie `sid` carrega 32 bytes aleatórios (base64url). O banco guarda só o **SHA-256** do token, de modo que um vazamento do banco não entrega sessões válidas. Expira em 7 dias. A escolha garante o requisito "token antigo não funciona após logout", que um JWT stateless não cumpre sem uma denylist.

### 6. Modelo de dados (Postgres 16 + Drizzle ORM / drizzle-kit migrations)
```
users                                   sessions
+------------------------------+        +-----------------------------+
| id          uuid PK          |<--+    | id_hash     text PK (sha256)|
| issuer      text             |   +----| user_id     uuid FK cascade |
| subject     text             |        | expires_at  timestamptz     |
| email       text null        |        | created_at  timestamptz     |
| name        text null        |        +-----------------------------+
| avatar_url  text null        |
| created_at / updated_at      |
| UNIQUE (issuer, subject)     |
+------------------------------+
```
A identidade é a chave (issuer, sub), não o email: o email pode mudar, e emissores diferentes podem repetir um `sub`. O login faz um upsert por essa chave. Postgres foi escolhido em vez de SQLite porque o app é multiusuário e o compose já existe; o Drizzle é tipado e leve.

### 7. Provedor fake: `ghcr.io/navikt/mock-oauth2-server`
É um OIDC real, com discovery, JWKS e tela de login interativa onde se digita o username, que vira o `sub`. Os testes usam `alice` e `bob`. Ele sobe no compose na porta `8080`, com `OIDC_ISSUER=http://localhost:8080/default`.

O compose roda **só a infra** (postgres + fake). API e web rodam no host via `npm run dev`. Isso evita divergência de issuer: o navegador e a API veem o mesmo `localhost:8080`, enquanto dentro da rede do compose a API veria outro hostname.

### 8. Trava de produção
Na inicialização, `AUTH_PROVIDER=fake` com `NODE_ENV=production` faz o processo sair com erro antes de abrir a porta. O atributo `Secure` dos cookies também depende de `NODE_ENV=production`.

### 9. Proteção de páginas no frontend
Um `AuthProvider` React chama `GET /api/me` ao carregar. Rotas protegidas redirecionam para `/login` em caso de 401, e `/login` redireciona para `/` se houver usuário. A proteção real dos dados fica na API: o front só cuida da navegação.

### 10. Testes
- **API (Vitest + `fastify.inject`)**: testes contra o Postgres do compose. O fluxo completo usa o fake OIDC real; o mock-oauth2-server aceita login não interativo, o que permite automatizar o callback. Os casos de erro (state, error, sem tx) não precisam do provedor.
- **E2E (Playwright)**: o navegador abre `/`, vai para `/login`, clica no botão, preenche `alice` no fake, volta para `/` com "Olá, alice", clica em "Sair" e volta para `/login`.

## Risks / Trade-offs

- [Login real com ChatGPT nunca testado] → Ficar em OIDC padrão via biblioteca certificada e manter o provedor só como configuração. Quando houver client ID, fazer um smoke test manual documentado no README.
- [SIWC pode exigir algo fora do OIDC padrão, como parâmetros extras ou só HTTPS no callback] → A configuração já aceita issuer e redirect arbitrários; um ajuste pontual, se necessário, entra numa change futura.
- [Fake OIDC aceita qualquer username] → A trava de produção e o rótulo visível "Fake OIDC" na página de login.
- [Cookie `auth_tx` limita a uma tentativa de login por navegador ao mesmo tempo] → É aceitável: uma nova tentativa sobrescreve a anterior.
- [Sessões expiradas se acumulam no banco] → A validação checa `expires_at`. A limpeza periódica fica para depois, já que o volume é baixo.

## Migration Plan

Não há migração: o projeto é greenfield. A primeira migration do drizzle-kit cria `users` e `sessions`. O rollback é apagar o volume do Postgres local.

## Open Questions

- Qual é a URL exata do issuer/discovery do SIWC e se ele aceita `http://localhost` no callback? Só importa quando houver client ID; entra como valor de configuração e não muda specs nem tarefas.
