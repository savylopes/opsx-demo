# opsx-demo: Todo List

Um todo list web multiusuário: cada pessoa entra com a própria conta e vê só as próprias tarefas.

O projeto também serve de demonstração do fluxo **OpenSpec / opsx**. Cada funcionalidade nasce como uma *change* (proposta, spec, design e tarefas), é implementada por um agente num worktree separado e só depois passa por avaliação, revisão de código, segurança e QA.

> **Estado atual:** em construção. A primeira change (`add-login`) está sendo implementada no branch `opsx/add-login`. Ainda não há código em `main`.

## O que o app faz

| Funcionalidade | Status |
|---|---|
| Login com provedor OpenID Connect (ChatGPT ou fake local) | em andamento (`add-login`) |
| Sessão, usuário atual e logout | em andamento (`add-login`) |
| Criar, listar, editar, concluir e remover tarefas | planejado (próxima change) |
| Filtros: todas / pendentes / concluídas | planejado |

Fora do escopo por enquanto: compartilhamento de listas, deploy em produção e app mobile.

## Arquitetura

```
  +-------------+   HTTP/JSON (/api)   +-------------+      SQL      +------------+
  |  Frontend   | -------------------> |    API      | ------------> | PostgreSQL |
  | React+Vite  | <------------------- |  Fastify    | <------------ |            |
  +-------------+   cookie de sessão   +------+------+               +------------+
                                              |
                                              | OpenID Connect (Authorization Code + PKCE)
                                              v
                                  +--------------------------+
                                  | Provedor OIDC            |
                                  |  - fake local (dev/test) |
                                  |  - ChatGPT (produção)    |
                                  +--------------------------+
```

- **Frontend:** React + Vite + TypeScript. Em desenvolvimento, o Vite faz proxy de `/api` para a API, então o navegador vê uma origem só.
- **API:** Node 22 + Fastify + TypeScript.
- **Banco:** PostgreSQL 16 com Drizzle ORM e migrations.
- **Testes:** Vitest (API e componentes) e Playwright (fluxo ponta a ponta).

## Autenticação

O login usa **OpenID Connect genérico**. O provedor é escolhido só por variáveis de ambiente, sem mudar código:

- **Fake local** (`mock-oauth2-server` em container): usado em desenvolvimento e nos testes. Você digita um nome de usuário (por exemplo `alice` ou `bob`) e entra. Ele **não pode** ser usado em produção: a API se recusa a iniciar com o fake quando `NODE_ENV=production`.
- **ChatGPT** ("Sign in with ChatGPT"): é o provedor de produção. Hoje a OpenAI só libera esse login para parceiros selecionados, e o projeto ainda não tem client ID. A integração fica pronta por configuração, mas ainda não foi testada com o ChatGPT de verdade.

Como funciona a sessão:

- O usuário é identificado pelo par (emissor, `sub`) do ID token, não pelo email.
- A sessão é guardada no servidor. O cookie `HttpOnly` leva só um token aleatório, e o banco guarda apenas o hash dele.
- O logout invalida a sessão de verdade: o token antigo deixa de funcionar.

## Como rodar

> Os comandos abaixo passam a funcionar quando a change `add-login` for integrada em `main`. Os detalhes completos de setup (variáveis de ambiente, usuários de teste, como trocar para o ChatGPT) chegam junto com ela.

Pré-requisitos: Node 22 e Docker.

```bash
docker compose up -d     # Postgres + provedor OIDC fake
npm install
npm run db:migrate
npm run dev              # API em :3000, web em http://localhost:5173
```

## Estrutura (planejada)

```
apps/
  web/                 frontend React
  api/                 API Fastify
docker-compose.yml     infraestrutura local (Postgres + OIDC fake)
openspec/
  specs/               o que o sistema faz hoje (specs consolidadas)
  changes/             mudanças em andamento (proposta, spec, design, tarefas)
```

## Fluxo de desenvolvimento (OpenSpec / opsx)

```
 /opsx:explore  -->  /opsx:propose  -->  /opsx-run <change> --validate  -->  /opsx-run <change> land
  pensar e          proposal, specs,     apply -> eval -> review ->           merge em main
  decidir           design, tasks        security -> qa (corrige até PASS)    + archive da change
```

1. **Explorar** (`/opsx:explore`): discutir a ideia e tomar as decisões.
2. **Propor** (`/opsx:propose`): gerar os artefatos da change em `openspec/changes/<change>/`.
3. **Aplicar e validar** (`/opsx-run <change> --validate`): numa janela tmux própria, o agente implementa no branch `opsx/<change>`. Em seguida outros agentes avaliam a spec, revisam o código, a segurança e a interface, e cada problema encontrado é corrigido.
4. **Integrar** (`/opsx-run <change> land`): faz o merge em `main` e arquiva a change, que passa a fazer parte de `openspec/specs/`.

Para ver uma change: `openspec show <change>`. Para listar as changes em andamento: `openspec list`.
