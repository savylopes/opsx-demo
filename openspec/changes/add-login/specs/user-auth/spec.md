# Spec Delta

## Purpose

Identifica o usuário do todo list por meio de um provedor OpenID Connect externo e mantém uma sessão autenticada no navegador, para que cada usuário acesse apenas os próprios dados.

## ADDED Requirements

### Requirement: Página de login
O sistema SHALL exibir, em `/login`, uma página com um botão "Entrar com <nome do provedor>", onde o nome é o do provedor OIDC configurado.

#### Scenario: Visitante abre a página de login
- **WHEN** um visitante sem sessão acessa `/login`
- **THEN** a página exibe o botão "Entrar com <nome do provedor configurado>"

#### Scenario: Erro de login é exibido
- **WHEN** o visitante é redirecionado para `/login` após uma tentativa de login que falhou
- **THEN** a página exibe uma mensagem de erro legível e o botão de login continua disponível

### Requirement: Início do fluxo de login
O sistema SHALL iniciar o login redirecionando o navegador ao provedor OIDC com o fluxo Authorization Code, PKCE (S256), `state` e `nonce` únicos por tentativa, e escopos `openid profile email`.

#### Scenario: Clique no botão de login
- **WHEN** o visitante aciona o botão de login, que leva a `GET /api/auth/login`
- **THEN** a resposta é um redirect para o endpoint de autorização do provedor contendo `response_type=code`, `code_challenge_method=S256`, `state`, `nonce` e `scope` com `openid profile email`

#### Scenario: Valores únicos por tentativa
- **WHEN** duas tentativas de login são iniciadas
- **THEN** cada uma recebe `state`, `nonce` e `code_challenge` diferentes

### Requirement: Callback de login bem-sucedido
Ao receber em `/api/auth/callback` um código válido com `state` correspondente à tentativa em andamento, o sistema SHALL trocar o código por tokens, validar o ID token (assinatura, emissor, audiência, expiração e nonce), abrir uma sessão e redirecionar para `/`.

#### Scenario: Primeiro login de um usuário
- **WHEN** o callback recebe um código válido de um usuário que nunca entrou
- **THEN** o sistema cria um usuário identificado pelo par (emissor, `sub`) com nome, email e foto do ID token, define o cookie de sessão e redireciona para `/`

#### Scenario: Login repetido não duplica o usuário
- **WHEN** o mesmo usuário (mesmo emissor e `sub`) entra novamente com nome ou email alterados no provedor
- **THEN** o sistema reutiliza o usuário existente, atualiza nome, email e foto, e não cria um segundo registro

### Requirement: Callback de login rejeitado
O sistema MUST recusar o callback sem abrir sessão quando o `state` estiver ausente, não corresponder à tentativa ou tiver expirado, quando o provedor retornar erro, ou quando o ID token for inválido. Nesses casos MUST redirecionar para `/login` com indicação de erro.

#### Scenario: State divergente
- **WHEN** o callback recebe um `state` diferente do emitido para aquele navegador
- **THEN** nenhuma sessão é criada e o navegador é redirecionado para `/login` com indicação de erro

#### Scenario: Provedor retorna erro
- **WHEN** o callback recebe o parâmetro `error` (por exemplo, o usuário negou o consentimento)
- **THEN** nenhuma sessão é criada e o navegador é redirecionado para `/login` com indicação de erro

#### Scenario: Callback sem tentativa em andamento
- **WHEN** o callback é acessado sem que um login tenha sido iniciado naquele navegador
- **THEN** nenhuma sessão é criada e o navegador é redirecionado para `/login` com indicação de erro

### Requirement: Cookie de sessão
O sistema SHALL representar a sessão com um cookie contendo um token opaco e aleatório, com os atributos `HttpOnly`, `SameSite=Lax` e `Path=/`, e também `Secure` quando executado em produção. A sessão MUST expirar em no máximo 7 dias.

#### Scenario: Atributos do cookie
- **WHEN** um login é concluído com sucesso
- **THEN** a resposta define o cookie de sessão com `HttpOnly`, `SameSite=Lax` e `Path=/`, e o valor não contém dados do usuário

#### Scenario: Sessão expirada
- **WHEN** uma requisição chega com um cookie de sessão cuja validade passou
- **THEN** o sistema trata a requisição como não autenticada

### Requirement: Usuário atual
O sistema SHALL expor `GET /api/me`. Com sessão válida, esse endpoint retorna o id, nome, email e URL da foto do usuário. Sem sessão válida, retorna 401.

#### Scenario: Com sessão
- **WHEN** um usuário logado chama `GET /api/me`
- **THEN** a resposta é 200 com `id`, `name`, `email` e `avatarUrl` do próprio usuário

#### Scenario: Sem sessão
- **WHEN** uma requisição sem cookie de sessão, ou com cookie inválido, chama `GET /api/me`
- **THEN** a resposta é 401 e não traz dados de nenhum usuário

### Requirement: Logout
O sistema SHALL expor `POST /api/auth/logout`, que invalida a sessão no servidor e remove o cookie. Depois do logout, o mesmo token de sessão MUST deixar de autenticar.

#### Scenario: Logout encerra a sessão
- **WHEN** um usuário logado aciona "Sair"
- **THEN** a sessão é invalidada, o cookie é removido e o navegador vai para `/login`

#### Scenario: Token antigo não funciona após logout
- **WHEN** uma requisição reapresenta o token de uma sessão encerrada por logout
- **THEN** `GET /api/me` responde 401

### Requirement: Proteção de páginas
O sistema SHALL redirecionar para `/login` o visitante sem sessão válida que acessa uma página protegida, como `/`, e SHALL redirecionar para `/` o usuário logado que acessa `/login`.

#### Scenario: Página inicial sem sessão
- **WHEN** um visitante sem sessão acessa `/`
- **THEN** ele é levado para `/login`

#### Scenario: Página inicial com sessão
- **WHEN** um usuário logado acessa `/`
- **THEN** a página exibe uma saudação com o nome dele e um botão "Sair"

#### Scenario: Login com sessão ativa
- **WHEN** um usuário logado acessa `/login`
- **THEN** ele é levado para `/`

### Requirement: Provedor fake apenas fora de produção
O sistema MUST se recusar a iniciar quando o provedor configurado é o fake local e o ambiente é produção.

#### Scenario: Fake em produção
- **WHEN** a API é iniciada em produção com o provedor fake configurado
- **THEN** a inicialização falha com uma mensagem que explica que o provedor fake não é permitido em produção

#### Scenario: Fake em desenvolvimento
- **WHEN** a API é iniciada em desenvolvimento ou teste com o provedor fake
- **THEN** a inicialização ocorre normalmente e o login funciona contra o provedor fake
