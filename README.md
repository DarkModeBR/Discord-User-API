# 📡 Discord Internal API — Documentação Não Oficial

> **⚠️ Aviso importante:** Esta documentação descreve a API utilizada internamente pelo client do Discord (web, desktop e mobile). Ela **não é a API oficial para bots** (`discord.com/developers`). Usar esta API com contas de usuário pode violar os [Termos de Serviço do Discord](https://discord.com/terms). Use apenas para fins educacionais e de estudo.

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Fundamentos — Headers & Autenticação](#2-fundamentos--headers--autenticação)
3. [Sistema de IDs — Snowflakes](#3-sistema-de-ids--snowflakes)
4. [Rate Limits](#4-rate-limits)
5. [Autenticação & Login](#5-autenticação--login)
6. [Usuários](#6-usuários)
7. [Mensagens](#7-mensagens)
8. [Canais](#8-canais)
9. [Servidores (Guilds)](#9-servidores-guilds)
10. [Relacionamentos (Amigos)](#10-relacionamentos-amigos)
11. [WebSocket / Gateway](#11-websocket--gateway)
12. [Eventos do Gateway](#12-eventos-do-gateway)
13. [CDN — Imagens & Assets](#13-cdn--imagens--assets)
14. [Códigos de Erro](#14-códigos-de-erro)

---

## 1. Visão Geral

### Base URL

```
https://discord.com/api/v9
```

O Discord usa a **versão 9** como padrão no client atual. A versão 10 também está disponível e é usada pela API oficial de bots.

| Versão | Status     | Observação                         |
|--------|------------|------------------------------------|
| v6     | Depreciada | Versão legada, ainda funcional     |
| v9     | Atual      | Usada pelo client oficial          |
| v10    | Atual      | Recomendada para bots              |

### Protocolos

- Toda comunicação usa **TLS 1.2+**
- REST: `https://discord.com/api/v9/`
- WebSocket (Gateway): `wss://gateway.discord.gg/?v=9&encoding=json`
- CDN: `https://cdn.discordapp.com/`

---

## 2. Fundamentos — Headers & Autenticação

Todo request HTTP ao client do Discord inclui um conjunto de headers obrigatórios e opcionais.

### Headers Obrigatórios

```http
Authorization: <token_do_usuario>
Content-Type: application/json
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 ...
X-Super-Properties: <base64_json>
```

### Header `X-Super-Properties`

Este é o header mais importante e específico do client. Ele contém um JSON **codificado em Base64** com metadados do client que o Discord usa para anti-abuso, A/B testing e feature gates.

**Estrutura do JSON (antes de encodar em Base64):**

```json
{
  "os": "Windows",
  "browser": "Chrome",
  "device": "",
  "system_locale": "pt-BR",
  "browser_user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...",
  "browser_version": "136.0.0.0",
  "os_version": "10",
  "referrer": "",
  "referring_domain": "",
  "referrer_current": "",
  "referring_domain_current": "",
  "release_channel": "stable",
  "client_build_number": 369171,
  "client_event_source": null
}
```

> **Importante:** O campo `client_build_number` deve ser recente. Muitas features experimentais verificam esse valor para liberar acesso. Um número desatualizado pode fazer certas rotas retornarem erros ou comportamentos inesperados.

**Exemplo para Desktop (Electron/Discord App):**

```json
{
  "os": "Windows",
  "browser": "Discord Client",
  "release_channel": "stable",
  "client_version": "1.0.328",
  "os_version": "10.0.22631",
  "os_arch": "x64",
  "system_locale": "pt-BR",
  "browser_user_agent": "Mozilla/5.0 ... discord/1.0.328 Chrome/134.0 Electron/35.1.5 Safari/537.36",
  "browser_version": "35.1.5",
  "client_build_number": 369171,
  "native_build_number": 56244,
  "client_event_source": null
}
```

**Gerando o header em JavaScript:**

```js
const props = {
  os: "Windows",
  browser: "Chrome",
  device: "",
  system_locale: "pt-BR",
  browser_user_agent: "Mozilla/5.0 ...",
  browser_version: "136.0.0.0",
  os_version: "10",
  release_channel: "stable",
  client_build_number: 369171,
  client_event_source: null
};

const encoded = btoa(JSON.stringify(props));
// Use como: X-Super-Properties: <encoded>
```

### Token de Usuário vs Token de Bot

| Tipo           | Prefixo no Header        | Acesso                                      |
|----------------|--------------------------|---------------------------------------------|
| User Token     | Sem prefixo (`mfa.xxx` ou `NDk...`) | Todas as rotas do usuário |
| Bot Token      | `Bot <token>`            | Rotas de bot (API oficial)                 |
| OAuth2 Bearer  | `Bearer <token>`         | Acesso escopado via OAuth2                 |

> **User tokens** são os tokens que o app do Discord usa internamente. Eles ficam armazenados no `localStorage` do browser (`token`) ou no `leveldb` do Electron.

---

## 3. Sistema de IDs — Snowflakes

O Discord usa o formato **Snowflake** do Twitter para todos os IDs (usuários, mensagens, canais, servidores, etc.).

### Estrutura de um Snowflake (64 bits)

```
63                                                   22          17     12        0
 +-------------------------------------------------+------------+------+----------+
 |          Timestamp (42 bits)                    | Worker ID  | PID  | Sequence |
 |  Milissegundos desde 01/01/2015 (Discord Epoch) |  (5 bits)  |(5b)  |  (12 b)  |
 +-------------------------------------------------+------------+------+----------+
```

- **Discord Epoch:** `1420070400000` (01/01/2015 em milissegundos)

### Extraindo o timestamp de um Snowflake

```js
function snowflakeToDate(snowflake) {
  const timestamp = BigInt(snowflake) >> 22n;
  return new Date(Number(timestamp) + 1420070400000);
}

snowflakeToDate("1234567890123456789");
// → Data de criação do objeto
```

### Gerando um Snowflake de um timestamp (para paginação)

```js
function dateToSnowflake(date) {
  const timestamp = BigInt(date.getTime()) - 1420070400000n;
  return String(timestamp << 22n);
}
```

> IDs são sempre retornados como **strings** no JSON da API para evitar overflow em linguagens que não suportam inteiros de 64 bits nativamente (como JavaScript).

---

## 4. Rate Limits

O Discord implementa rate limiting por rota e globalmente, seguindo a RFC 6585.

### Headers de Rate Limit (resposta)

```http
X-RateLimit-Limit: 5
X-RateLimit-Remaining: 3
X-RateLimit-Reset: 1470173023
X-RateLimit-Reset-After: 1.5
X-RateLimit-Bucket: abcd1234
X-RateLimit-Global: true       (apenas em 429 global)
X-RateLimit-Scope: user        (apenas em 429)
```

| Header                  | Descrição                                                   |
|-------------------------|-------------------------------------------------------------|
| `X-RateLimit-Limit`     | Máximo de requests no bucket atual                         |
| `X-RateLimit-Remaining` | Requests restantes no bucket atual                         |
| `X-RateLimit-Reset`     | Epoch (segundos) em que o limite reseta                    |
| `X-RateLimit-Reset-After` | Segundos até o reset (com decimais para precisão)       |
| `X-RateLimit-Bucket`    | Identificador único do bucket de rate limit                |
| `X-RateLimit-Global`    | Indica rate limit global (retornado apenas no HTTP 429)    |

### Resposta HTTP 429 (Too Many Requests)

```json
{
  "message": "You are being rate limited.",
  "retry_after": 1.337,
  "global": false
}
```

### Boas práticas

- **Nunca hard-code rate limits** — eles mudam dinamicamente
- Sempre leia o header `Retry-After` antes de retentar
- Ignorar rate limits de forma persistente resulta em **ban de IP ou conta**
- Implemente **jitter aleatório** em automações para evitar padrões detectáveis

---

## 5. Autenticação & Login

### Login com Email/Senha

```http
POST /auth/login
Content-Type: application/json

{
  "login": "email@exemplo.com",
  "password": "senha123",
  "undelete": false,
  "captcha_key": null,
  "login_source": null,
  "gift_code_sku_id": null
}
```

**Resposta (sucesso):**

```json
{
  "user_id": "123456789012345678",
  "token": "NDk....<token_aqui>",
  "user_settings": {
    "locale": "pt-BR",
    "theme": "dark"
  }
}
```

**Resposta (MFA necessária):**

```json
{
  "mfa": true,
  "sms": false,
  "ticket": "eyJhbGc...",
  "backup": false,
  "totp": true,
  "webauthn": null
}
```

### Verificação MFA (TOTP)

```http
POST /auth/mfa/totp
Content-Type: application/json

{
  "code": "123456",
  "ticket": "eyJhbGc...",
  "login_source": null,
  "gift_code_sku_id": null
}
```

**Resposta:**

```json
{
  "token": "NDk....<token_aqui>"
}
```

### Logout

```http
POST /auth/logout
Authorization: <token>
Content-Type: application/json

{
  "provider": null,
  "voip_provider": null
}
```

### Remote Auth (Login via QR Code)

O client desktop usa WebSocket para o fluxo de QR code:

```
wss://remote-auth-gateway.discord.gg/?v=2
```

**Fluxo:**
1. Client conecta ao WebSocket de remote auth
2. Recebe `op: hello` com o heartbeat interval
3. Gera keypair RSA-2048 e envia `op: init` com a chave pública
4. Recebe `op: nonce_proof` — deve responder com o nonce decifrado
5. Recebe `op: pending_remote_init` com o **fingerprint**
6. Fingerprint é encodado em QR: `https://discord.com/ra/<fingerprint>`
7. Ao escanear no mobile, recebe `op: pending_finish` com dados do usuário (cifrados)
8. Ao confirmar no mobile, recebe `op: finish` com o token de autenticação

---

## 6. Usuários

### Obter o próprio perfil

```http
GET /users/@me
Authorization: <token>
```

**Resposta:**

```json
{
  "id": "123456789012345678",
  "username": "usuario",
  "discriminator": "0",
  "global_name": "Nome Global",
  "avatar": "a_hash_do_avatar",
  "banner": "hash_do_banner",
  "accent_color": 16711680,
  "email": "email@exemplo.com",
  "verified": true,
  "mfa_enabled": false,
  "premium_type": 2,
  "flags": 0
}
```

### Obter perfil de outro usuário

```http
GET /users/{user_id}
Authorization: <token>
```

### Obter perfil completo (com mutual friends, guilds, etc.)

```http
GET /users/{user_id}/profile?with_mutual_guilds=true&with_mutual_friends=true
Authorization: <token>
```

### Atualizar o próprio perfil

```http
PATCH /users/@me
Authorization: <token>
Content-Type: application/json

{
  "username": "NovoNome",
  "avatar": "data:image/png;base64,<base64_da_imagem>",
  "banner": "data:image/png;base64,<base64_do_banner>",
  "bio": "Minha bio aqui",
  "accent_color": 16711680
}
```

### Configurações do usuário (proto-encoded)

O Discord migrou as configurações para o formato **proto binário**. O endpoint é:

```http
GET /users/@me/settings-proto/1     ← Configurações de usuário
GET /users/@me/settings-proto/2     ← Configurações de frequência
GET /users/@me/settings-proto/3     ← Pasta de guilds
PATCH /users/@me/settings-proto/1
```

---

## 7. Mensagens

### Buscar mensagens de um canal

```http
GET /channels/{channel_id}/messages?limit=50
Authorization: <token>
```

**Parâmetros de query:**

| Parâmetro | Tipo      | Descrição                                          |
|-----------|-----------|----------------------------------------------------|
| `limit`   | integer   | 1–100, default 50                                  |
| `before`  | snowflake | Mensagens anteriores a este ID                    |
| `after`   | snowflake | Mensagens posteriores a este ID                   |
| `around`  | snowflake | Mensagens em torno deste ID (mutuamente exclusivos)|

### Enviar mensagem

```http
POST /channels/{channel_id}/messages
Authorization: <token>
Content-Type: application/json

{
  "content": "Olá, mundo!",
  "tts": false,
  "nonce": "1234567890",
  "allowed_mentions": {
    "parse": ["users", "roles"],
    "replied_user": false
  }
}
```

**Com reply:**

```json
{
  "content": "Respondendo!",
  "message_reference": {
    "message_id": "987654321098765432",
    "channel_id": "123456789012345678",
    "guild_id": "111111111111111111"
  }
}
```

**Com embed:**

```json
{
  "content": "",
  "embeds": [{
    "title": "Título",
    "description": "Descrição do embed",
    "color": 5763719,
    "fields": [
      { "name": "Campo 1", "value": "Valor 1", "inline": true }
    ],
    "footer": { "text": "Rodapé" },
    "timestamp": "2025-01-01T00:00:00.000Z"
  }]
}
```

### Editar mensagem

```http
PATCH /channels/{channel_id}/messages/{message_id}
Authorization: <token>
Content-Type: application/json

{
  "content": "Mensagem editada"
}
```

### Deletar mensagem

```http
DELETE /channels/{channel_id}/messages/{message_id}
Authorization: <token>
```

### Deletar múltiplas mensagens (bulk delete)

```http
POST /channels/{channel_id}/messages/bulk-delete
Authorization: <token>
Content-Type: application/json

{
  "messages": ["id1", "id2", "id3"]
}
```

> **Limite:** Máximo de 100 mensagens por vez. Mensagens com mais de 14 dias não podem ser apagadas em bulk.

### Adicionar reação

```http
PUT /channels/{channel_id}/messages/{message_id}/reactions/{emoji}/@me
Authorization: <token>
```

O emoji pode ser:
- Unicode: `👍` → encode como `%F0%9F%91%8D`
- Custom: `nome:id` → ex: `pepe:123456789`

### Buscar mensagem específica

```http
GET /channels/{channel_id}/messages/{message_id}
Authorization: <token>
```

### Marcar como lida (ACK)

```http
POST /channels/{channel_id}/messages/{message_id}/ack
Authorization: <token>
Content-Type: application/json

{
  "token": null
}
```

---

## 8. Canais

### Obter canal

```http
GET /channels/{channel_id}
Authorization: <token>
```

### Modificar canal

```http
PATCH /channels/{channel_id}
Authorization: <token>
Content-Type: application/json

{
  "name": "novo-nome",
  "topic": "Novo tópico do canal",
  "nsfw": false,
  "rate_limit_per_user": 5,
  "position": 0
}
```

### Deletar / fechar canal

```http
DELETE /channels/{channel_id}
Authorization: <token>
```

### Iniciar digitação (typing indicator)

```http
POST /channels/{channel_id}/typing
Authorization: <token>
```

> O indicador de "está digitando..." dura ~10 segundos. Envie a cada 8s enquanto o usuário digitar.

### Mensagens fixadas (pinned)

```http
GET /channels/{channel_id}/pins
PUT /channels/{channel_id}/pins/{message_id}
DELETE /channels/{channel_id}/pins/{message_id}
```

### Criar DM

```http
POST /users/@me/channels
Authorization: <token>
Content-Type: application/json

{
  "recipient_id": "123456789012345678"
}
```

### Criar grupo de DM

```http
POST /users/@me/channels
Authorization: <token>
Content-Type: application/json

{
  "recipients": ["id1", "id2", "id3"]
}
```

### Threads

```http
// Criar thread a partir de mensagem
POST /channels/{channel_id}/messages/{message_id}/threads

// Criar thread standalone (em forum ou texto)
POST /channels/{channel_id}/threads

// Listar threads arquivadas
GET /channels/{channel_id}/threads/archived/public
GET /channels/{channel_id}/threads/archived/private

// Entrar/sair de thread
PUT /channels/{channel_id}/thread-members/@me
DELETE /channels/{channel_id}/thread-members/@me
```

---

## 9. Servidores (Guilds)

### Obter informações do servidor

```http
GET /guilds/{guild_id}
GET /guilds/{guild_id}?with_counts=true    ← inclui contagem de membros
Authorization: <token>
```

### Listar canais do servidor

```http
GET /guilds/{guild_id}/channels
Authorization: <token>
```

### Listar membros

```http
GET /guilds/{guild_id}/members?limit=1000&after=0
Authorization: <token>
```

### Buscar membros por nome

```http
GET /guilds/{guild_id}/members/search?query=nome&limit=10
Authorization: <token>
```

### Obter membro específico

```http
GET /guilds/{guild_id}/members/{user_id}
Authorization: <token>
```

### Modificar membro

```http
PATCH /guilds/{guild_id}/members/{user_id}
Authorization: <token>
Content-Type: application/json

{
  "nick": "Novo apelido",
  "roles": ["role_id_1", "role_id_2"],
  "mute": false,
  "deaf": false
}
```

### Entrar no servidor (via invite)

```http
POST /invites/{invite_code}
Authorization: <token>
Content-Type: application/json

{}
```

### Sair do servidor

```http
DELETE /users/@me/guilds/{guild_id}
Authorization: <token>
```

### Cargos (Roles)

```http
GET /guilds/{guild_id}/roles                       ← listar
POST /guilds/{guild_id}/roles                      ← criar
PATCH /guilds/{guild_id}/roles/{role_id}           ← editar
DELETE /guilds/{guild_id}/roles/{role_id}          ← deletar
PATCH /guilds/{guild_id}/roles                     ← reordenar (array de {id, position})
```

### Bans

```http
GET /guilds/{guild_id}/bans                        ← listar banidos
GET /guilds/{guild_id}/bans/{user_id}              ← ver ban específico
PUT /guilds/{guild_id}/bans/{user_id}              ← banir usuário
DELETE /guilds/{guild_id}/bans/{user_id}           ← desbanir
```

### Convites do servidor

```http
GET /guilds/{guild_id}/invites
```

### Audit Log

```http
GET /guilds/{guild_id}/audit-logs?limit=50&action_type=1
```

**Tipos de ação (`action_type`) comuns:**

| ID | Ação                    |
|----|-------------------------|
| 1  | Guild Update            |
| 10 | Channel Create          |
| 11 | Channel Update          |
| 12 | Channel Delete          |
| 20 | Member Kick             |
| 21 | Member Prune            |
| 22 | Member Ban Add          |
| 23 | Member Ban Remove       |
| 24 | Member Update           |
| 25 | Member Role Update      |
| 30 | Role Create             |
| 31 | Role Update             |
| 32 | Role Delete             |
| 72 | Message Delete          |
| 74 | Message Bulk Delete     |

---

## 10. Relacionamentos (Amigos)

### Listar relacionamentos

```http
GET /users/@me/relationships
Authorization: <token>
```

**Resposta:**

```json
[
  {
    "id": "123456789012345678",
    "type": 1,
    "nickname": null,
    "user": {
      "id": "123456789012345678",
      "username": "amigo",
      "discriminator": "0",
      "avatar": "hash"
    }
  }
]
```

**Tipos de relacionamento:**

| Tipo | Significado         |
|------|---------------------|
| 1    | Amigo               |
| 2    | Bloqueado           |
| 3    | Pedido recebido     |
| 4    | Pedido enviado      |
| 5    | Implícito (mesmo servidor) |

### Enviar pedido de amizade

```http
PUT /users/@me/relationships/{user_id}
Authorization: <token>
Content-Type: application/json

{
  "type": 1
}
```

### Aceitar pedido de amizade

```http
PUT /users/@me/relationships/{user_id}
Authorization: <token>
Content-Type: application/json

{}
```

### Remover amigo / Desbloquear / Recusar pedido

```http
DELETE /users/@me/relationships/{user_id}
Authorization: <token>
```

### Bloquear usuário

```http
PUT /users/@me/relationships/{user_id}
Authorization: <token>
Content-Type: application/json

{
  "type": 2
}
```

---

## 11. WebSocket / Gateway

O Gateway é a conexão WebSocket persistente que o client usa para receber eventos em tempo real.

### Conectando ao Gateway

**Passo 1:** Obtenha a URL do Gateway

```http
GET /gateway
```

Resposta: `{ "url": "wss://gateway.discord.gg" }`

**Passo 2:** Conecte via WebSocket

```
wss://gateway.discord.gg/?v=9&encoding=json
```

Parâmetros opcionais:
- `encoding=etf` — usa Erlang Term Format (mais eficiente)
- `compress=zlib-stream` — habilita compressão zlib

### Estrutura do Payload

```json
{
  "op": 0,
  "d": { ... },
  "s": 42,
  "t": "MESSAGE_CREATE"
}
```

| Campo | Tipo    | Descrição                                            |
|-------|---------|------------------------------------------------------|
| `op`  | integer | Opcode do payload                                    |
| `d`   | object  | Dados do evento (pode ser `null`)                   |
| `s`   | integer | Número de sequência (apenas em op 0, para heartbeat) |
| `t`   | string  | Nome do evento (apenas em op 0)                     |

### Tabela de Opcodes

| OP | Nome             | Direção        | Descrição                                            |
|----|------------------|----------------|------------------------------------------------------|
| 0  | Dispatch         | Servidor→Client | Evento de dispatch (maioria dos eventos)            |
| 1  | Heartbeat        | Ambos          | Ping para manter conexão viva                        |
| 2  | Identify         | Client→Servidor | Autenticação inicial                                |
| 3  | Presence Update  | Client→Servidor | Atualizar status/presença                           |
| 4  | Voice State Update | Client→Servidor | Entrar/sair de canal de voz                       |
| 6  | Resume           | Client→Servidor | Retomar sessão após desconexão                      |
| 7  | Reconnect        | Servidor→Client | Servidor pede reconexão                             |
| 8  | Request Guild Members | Client→Servidor | Solicitar lista de membros                    |
| 9  | Invalid Session  | Servidor→Client | Sessão inválida, re-identificar necessário          |
| 10 | Hello            | Servidor→Client | Primeiro payload, com heartbeat_interval            |
| 11 | Heartbeat ACK    | Servidor→Client | Confirmação de heartbeat recebido                   |

### Fluxo completo de conexão

```
Client                          Gateway
  |                                 |
  |──── WebSocket Connect ─────────>|
  |                                 |
  |<─── OP 10 Hello ───────────────|
  |   { heartbeat_interval: 41250 } |
  |                                 |
  |──── OP 1 Heartbeat ────────────>|   ← Enviar imediatamente
  |   { "op": 1, "d": null }        |
  |                                 |
  |<─── OP 11 Heartbeat ACK ───────|
  |                                 |
  |──── OP 2 Identify ─────────────>|
  |   (ver estrutura abaixo)        |
  |                                 |
  |<─── OP 0 READY ────────────────|
  |   (dados iniciais da sessão)    |
  |                                 |
  |<─── OP 0 GUILD_CREATE ─────────|   ← Múltiplos, um por servidor
  |                                 |
  |  ... eventos em tempo real ...  |
  |                                 |
  |──── OP 1 Heartbeat ────────────>|   ← A cada heartbeat_interval ms
```

### OP 2 — Identify (User Client)

```json
{
  "op": 2,
  "d": {
    "token": "<seu_token>",
    "capabilities": 30717,
    "properties": {
      "os": "Windows",
      "browser": "Chrome",
      "device": "",
      "system_locale": "pt-BR",
      "browser_user_agent": "Mozilla/5.0 ...",
      "browser_version": "136.0.0.0",
      "os_version": "10",
      "referrer": "",
      "referring_domain": "",
      "referrer_current": "",
      "referring_domain_current": "",
      "release_channel": "stable",
      "client_build_number": 369171,
      "client_event_source": null
    },
    "presence": {
      "status": "online",
      "since": 0,
      "activities": [],
      "afk": false
    },
    "compress": false,
    "client_state": {
      "guild_versions": {},
      "highest_last_message_id": "0",
      "read_state_version": 0,
      "user_guild_settings_version": -1,
      "user_settings_version": -1,
      "private_channels_version": "0",
      "api_code_version": 0
    }
  }
}
```

> O campo `capabilities` é uma **bitmask** que indica quais recursos o client suporta. O valor `30717` é o usado pelo client web atual.

### OP 3 — Presence/Status Update

```json
{
  "op": 3,
  "d": {
    "status": "online",
    "since": 0,
    "afk": false,
    "activities": [
      {
        "name": "Visual Studio Code",
        "type": 0,
        "details": "Editando arquivo.py",
        "state": "No workspace"
      }
    ]
  }
}
```

**Tipos de status:** `online`, `idle`, `dnd`, `invisible`

**Tipos de atividade (`type`):**

| Tipo | Nome      | Exibição             |
|------|-----------|----------------------|
| 0    | Playing   | "Jogando X"          |
| 1    | Streaming | "Transmitindo X"     |
| 2    | Listening | "Ouvindo X"          |
| 3    | Watching  | "Assistindo X"       |
| 4    | Custom    | Status personalizado |
| 5    | Competing | "Competindo em X"    |

### OP 6 — Resume (Retomar sessão)

```json
{
  "op": 6,
  "d": {
    "token": "<seu_token>",
    "session_id": "abc123...",
    "seq": 1337
  }
}
```

### Heartbeat

O client deve enviar um heartbeat a cada `heartbeat_interval` milissegundos:

```json
{
  "op": 1,
  "d": 1337
}
```

onde `d` é o último número de sequência recebido (campo `s` do último OP 0), ou `null` se nenhum foi recebido.

Se o servidor não responder com OP 11 antes do próximo heartbeat, feche a conexão e reconecte.

---

## 12. Eventos do Gateway

Os eventos chegam como OP 0 com o campo `t` indicando o tipo.

### Eventos de Mensagem

| Evento               | Descrição                          |
|----------------------|------------------------------------|
| `MESSAGE_CREATE`     | Nova mensagem enviada              |
| `MESSAGE_UPDATE`     | Mensagem editada                   |
| `MESSAGE_DELETE`     | Mensagem deletada                  |
| `MESSAGE_DELETE_BULK`| Múltiplas mensagens deletadas      |
| `MESSAGE_REACTION_ADD` | Reação adicionada               |
| `MESSAGE_REACTION_REMOVE` | Reação removida             |
| `TYPING_START`       | Usuário começou a digitar          |

### Eventos de Presença

| Evento             | Descrição                            |
|--------------------|--------------------------------------|
| `PRESENCE_UPDATE`  | Status/atividade de usuário mudou    |

### Eventos de Canais

| Evento              | Descrição                          |
|---------------------|------------------------------------|
| `CHANNEL_CREATE`    | Canal criado                       |
| `CHANNEL_UPDATE`    | Canal atualizado                   |
| `CHANNEL_DELETE`    | Canal deletado                     |
| `THREAD_CREATE`     | Thread criada                      |
| `THREAD_UPDATE`     | Thread atualizada                  |
| `THREAD_DELETE`     | Thread deletada                    |

### Eventos de Servidor

| Evento               | Descrição                          |
|----------------------|------------------------------------|
| `GUILD_CREATE`       | Servidor carregado/entrado         |
| `GUILD_UPDATE`       | Servidor atualizado                |
| `GUILD_DELETE`       | Servidor deletado / saiu           |
| `GUILD_MEMBER_ADD`   | Membro entrou                      |
| `GUILD_MEMBER_UPDATE`| Membro atualizado                  |
| `GUILD_MEMBER_REMOVE`| Membro saiu/foi removido           |
| `GUILD_ROLE_CREATE`  | Cargo criado                       |
| `GUILD_ROLE_UPDATE`  | Cargo atualizado                   |
| `GUILD_ROLE_DELETE`  | Cargo deletado                     |
| `GUILD_BAN_ADD`      | Usuário banido                     |
| `GUILD_BAN_REMOVE`   | Ban removido                       |

### Evento READY

O evento mais importante — enviado após o Identify bem-sucedido:

```json
{
  "t": "READY",
  "op": 0,
  "d": {
    "v": 9,
    "user": { "id": "...", "username": "...", ... },
    "guilds": [ { "id": "...", "unavailable": true }, ... ],
    "session_id": "abc123...",
    "resume_gateway_url": "wss://gateway-us-east1-b.discord.gg",
    "shard": [0, 1],
    "application": { "id": "...", "flags": 0 }
  }
}
```

> Salve o `session_id` e o `resume_gateway_url` para poder reconectar com OP 6 Resume.

### Códigos de fechamento WebSocket

| Código | Pode reconectar? | Descrição                              |
|--------|------------------|----------------------------------------|
| 4000   | ✅               | Erro desconhecido                      |
| 4001   | ✅               | Opcode desconhecido                    |
| 4002   | ✅               | Payload inválido                       |
| 4003   | ✅               | Não autenticado                        |
| 4004   | ❌               | Token inválido                         |
| 4005   | ✅               | Já autenticado                         |
| 4007   | ✅               | Sequência inválida no Resume           |
| 4008   | ✅               | Rate limited                           |
| 4009   | ✅               | Sessão expirada                        |
| 4010   | ❌               | Shard inválido                         |
| 4011   | ❌               | Sharding necessário                    |
| 4012   | ❌               | Versão da API inválida                 |
| 4013   | ❌               | Intents inválidos                      |
| 4014   | ❌               | Intents não permitidos                 |

---

## 13. CDN — Imagens & Assets

Base URL: `https://cdn.discordapp.com`

### Formatos disponíveis

| Formato | Extensão | Suporte a animação |
|---------|----------|--------------------|
| PNG     | `.png`   | ❌                 |
| JPEG    | `.jpg`   | ❌                 |
| WebP    | `.webp`  | ✅ (animado)       |
| GIF     | `.gif`   | ✅                 |

Tamanhos disponíveis (parâmetro `?size=`): `16, 32, 64, 128, 256, 512, 1024, 2048, 4096`

### Endpoints CDN

```
# Avatar de usuário
https://cdn.discordapp.com/avatars/{user_id}/{avatar_hash}.png?size=256

# Avatar animado (hash começa com "a_")
https://cdn.discordapp.com/avatars/{user_id}/a_{hash}.gif?size=256

# Avatar padrão (sem avatar customizado)
https://cdn.discordapp.com/embed/avatars/{(user_id >> 22) % 6}.png

# Banner de usuário
https://cdn.discordapp.com/banners/{user_id}/{banner_hash}.png?size=600

# Ícone do servidor
https://cdn.discordapp.com/icons/{guild_id}/{icon_hash}.png?size=256

# Banner do servidor
https://cdn.discordapp.com/banners/{guild_id}/{banner_hash}.png?size=600

# Splash do servidor (tela de boas-vindas)
https://cdn.discordapp.com/splashes/{guild_id}/{splash_hash}.png?size=2048

# Emoji customizado
https://cdn.discordapp.com/emojis/{emoji_id}.png
https://cdn.discordapp.com/emojis/{emoji_id}.gif    ← animado

# Sticker
https://cdn.discordapp.com/stickers/{sticker_id}.png
https://cdn.discordapp.com/stickers/{sticker_id}.gif

# Ícone de aplicação
https://cdn.discordapp.com/app-icons/{app_id}/{icon_hash}.png

# Asset de aplicação
https://cdn.discordapp.com/app-assets/{app_id}/{asset_id}.png

# Attachment de mensagem
https://cdn.discordapp.com/attachments/{channel_id}/{attachment_id}/{filename}
```

> **Dica:** Hashes que começam com `a_` indicam que o asset está disponível em formato animado.

---

## 14. Códigos de Erro

### Erros HTTP

| Código | Significado                              |
|--------|------------------------------------------|
| 200    | OK                                       |
| 201    | Created                                  |
| 204    | No Content                               |
| 304    | Not Modified                             |
| 400    | Bad Request                              |
| 401    | Unauthorized (token inválido/ausente)    |
| 403    | Forbidden (sem permissão)                |
| 404    | Not Found                                |
| 405    | Method Not Allowed                       |
| 429    | Too Many Requests (rate limited)         |
| 502    | Gateway Unavailable                      |
| 5xx    | Erro interno do Discord                  |

### Erros JSON da API

```json
{
  "code": 50013,
  "message": "Missing Permissions",
  "errors": {
    "content": {
      "_errors": [
        { "code": "BASE_TYPE_MAX_LENGTH", "message": "Must be 4000 or fewer in length." }
      ]
    }
  }
}
```

### Códigos de Erro Comuns

| Código  | Mensagem                              |
|---------|---------------------------------------|
| 0       | General error                         |
| 10001   | Unknown Account                       |
| 10002   | Unknown Application                   |
| 10003   | Unknown Channel                       |
| 10004   | Unknown Guild                         |
| 10006   | Unknown Invite                        |
| 10007   | Unknown Member                        |
| 10008   | Unknown Message                       |
| 10011   | Unknown Role                          |
| 10013   | Unknown User                          |
| 20001   | Bots cannot use this endpoint         |
| 20002   | Only bots can use this endpoint       |
| 40001   | Unauthorized                          |
| 40002   | Verification required                 |
| 40007   | User is banned from the guild         |
| 50001   | Missing Access                        |
| 50007   | Cannot send messages to this user     |
| 50013   | Missing Permissions                   |
| 50035   | Invalid Form Body                     |
| 90001   | Reaction blocked                      |
| 130000  | API resource is currently overloaded  |

---

## Referências & Recursos

- [Discord Userdoccers](https://docs.discord.food) — Documentação não oficial detalhada
- [Luna's Unofficial Docs](https://luna.gitlab.io/discord-unofficial-docs/) — Documentação de endpoints ocultos
- [Discord API Endpoints (Gist)](https://gist.github.com/hackermondev/5c928ca12b4f4e6320100b11f798c23b) — Lista completa de endpoints do client v9

---

*Documentação gerada para fins educacionais. Última atualização: Março 2026.*
