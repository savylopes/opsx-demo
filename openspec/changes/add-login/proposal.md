# Proposal

## Why

O projeto será um todo list web multiusuário (frontend + API + banco), em que cada usuário só vê as próprias tarefas. Antes de qualquer tarefa existir, o sistema precisa saber quem é o usuário. Esta change entrega a página de login e a fundação de autenticação sobre a qual o CRUD de tarefas será construído.

O login desejado é "Sign in with ChatGPT" (OpenID Connect). Hoje esse login só está disponível para parceiros selecionados, e o projeto ainda não tem client ID. Por isso a autenticação é feita sobre OIDC genérico: em desenvolvimento e nos testes usa um provedor OIDC fake local, e o ChatGPT entra por configuração quando houver credenciais.

## What Changes

- Fundação do projeto, que hoje é um repositório vazio: frontend web, API HTTP e banco de dados, com ambiente local subindo por um comando só.
- Página de login com um botão "Entrar com <provedor>" que inicia o fluxo OIDC (Authorization Code + PKCE).
- Callback que valida a resposta do provedor, cria ou atualiza o usuário e abre uma sessão em cookie.
- Endpoint para consultar o usuário logado e logout que invalida a sessão no servidor.
- Proteção de páginas: visitante sem sessão é levado ao login; usuário logado que abre o login é levado à página inicial.
- Página inicial mínima, só com saudação e botão de sair. As tarefas ficam para a próxima change.
- Provedor OIDC fake local para desenvolvimento e testes automatizados, bloqueado em produção.

Premissas desta proposta (padrões propostos na exploração e ainda não confirmados de forma explícita):
- Escopo: o CRUD de tarefas, compartilhamento de listas e deploy em produção ficam **fora** desta change.
- Stack: TypeScript no front e no back, banco relacional (ver design.md).

## Capabilities

### New Capabilities
- `user-auth`: identificação do usuário via provedor OpenID Connect externo: página de login, callback, sessão, usuário atual, logout, proteção de páginas e restrições do provedor fake.

### Modified Capabilities
<!-- Nenhuma: o projeto não tem specs existentes. -->

## Impact

- **Código:** cria o projeto do zero (frontend, API, migrations do banco, configuração do ambiente local).
- **APIs:** novos endpoints `/api/auth/login`, `/api/auth/callback`, `/api/auth/logout` e `/api/me`.
- **Dados:** novas tabelas de usuários e de sessões.
- **Dependências externas:** um provedor OIDC. Em dev/teste é o fake local em container; em produção será o ChatGPT (SIWC), que depende de um client ID ainda não obtido.
- **Risco conhecido:** o login real com ChatGPT não vai ser validado nesta change, porque fica pronto apenas por configuração.
