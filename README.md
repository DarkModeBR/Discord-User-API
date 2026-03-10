# 📡 Discord Internal API — Documentação Não Oficial

> **⚠️ Aviso importante:** Esta documentação descreve a API utilizada internamente pelo client do Discord (web, desktop e mobile). Ela **não é a API oficial para bots** (`discord.com/developers`). Usar esta API com contas de usuário pode violar os [Termos de Serviço do Discord](https://discord.com/terms). Use apenas para fins educacionais e de estudo.

---

## Sumário

1. [Visão Geral](#1-visão-geral)
2. [Fundamentos — Headers & Autenticação](#2-fundamentos--headers--autenticação)
3. [Autenticação & Login](#3-autenticação--login)
4. [Usuários](#4-usuários)
5. [Mensagens](#5-mensagens)
6. [Canais](#6-canais)
7. [Servidores (Guilds)](#7-servidores-guilds)
8. [Relacionamentos (Amigos)](#8-relacionamentos-amigos)
9. [CDN — Imagens & Assets](#9-cdn--imagens--assets)
10. [Webhooks](#10-webhooks)
11. [Busca de Mensagens](#11-busca-de-mensagens)
12. [Convites (Invites)](#12-convites-invites)
13. [Formatação de Texto (Markdown)](#13-formatação-de-texto-markdown)

---

## 1. Visão Geral

### Base URL

```
https://discord.com/api/v9
```

O Discord usa a **versão 9** como padrão no client atual. A versão 10 também está disponível e é usada pela API oficial de bots.

| Versão | Status     | Observação                      |
|--------|------------|---------------------------------|
| v6     | Depreciada | Versão legada, ainda funcional  |
| v9     | Atual      | Usada pelo client oficial       |
| v10    | Atual      | Recomendada para bots           |

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

| Tipo          | Prefixo no Header                   | Acesso                            |
|---------------|-------------------------------------|-----------------------------------|
| User Token    | Sem prefixo (`mfa.xxx` ou `NDk...`) | Todas as rotas do usuário         |
| Bot Token     | `Bot <token>`                       | Rotas de bot (API oficial)        |
| OAuth2 Bearer | `Bearer <token>`                    | Acesso escopado via OAuth2        |

> **User tokens** são os tokens que o app do Discord usa internamente. Eles ficam armazenados no `localStorage` do browser (`token`) ou no `leveldb` do Electron.

---

## 3. Autenticação & Login

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

---

## 4. Usuários

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

---

## 5. Mensagens

### Buscar mensagens de um canal

```http
GET /channels/{channel_id}/messages?limit=50
Authorization: <token>
```

**Parâmetros de query:**

| Parâmetro | Tipo      | Descrição                                           |
|-----------|-----------|-----------------------------------------------------|
| `limit`   | integer   | 1–100, default 50                                   |
| `before`  | snowflake | Mensagens anteriores a este ID                      |
| `after`   | snowflake | Mensagens posteriores a este ID                     |
| `around`  | snowflake | Mensagens em torno deste ID (mutuamente exclusivos) |

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

## 6. Canais

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

---

## 7. Servidores (Guilds)

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
GET /guilds/{guild_id}/roles                    ← listar
POST /guilds/{guild_id}/roles                   ← criar
PATCH /guilds/{guild_id}/roles/{role_id}        ← editar
DELETE /guilds/{guild_id}/roles/{role_id}       ← deletar
PATCH /guilds/{guild_id}/roles                  ← reordenar (array de {id, position})
```

### Bans

```http
GET /guilds/{guild_id}/bans                     ← listar banidos
GET /guilds/{guild_id}/bans/{user_id}           ← ver ban específico
PUT /guilds/{guild_id}/bans/{user_id}           ← banir usuário
DELETE /guilds/{guild_id}/bans/{user_id}        ← desbanir
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

| ID | Ação                |
|----|---------------------|
| 1  | Guild Update        |
| 10 | Channel Create      |
| 11 | Channel Update      |
| 12 | Channel Delete      |
| 20 | Member Kick         |
| 21 | Member Prune        |
| 22 | Member Ban Add      |
| 23 | Member Ban Remove   |
| 24 | Member Update       |
| 25 | Member Role Update  |
| 30 | Role Create         |
| 31 | Role Update         |
| 32 | Role Delete         |
| 72 | Message Delete      |
| 74 | Message Bulk Delete |

---

## 8. Relacionamentos (Amigos)

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

| Tipo | Significado              |
|------|--------------------------|
| 1    | Amigo                    |
| 2    | Bloqueado                |
| 3    | Pedido recebido          |
| 4    | Pedido enviado           |
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

## 9. CDN — Imagens & Assets

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

## 10. Webhooks

### Criar webhook em um canal

```http
POST /channels/{channel_id}/webhooks
Authorization: <token>
Content-Type: application/json

{
  "name": "Meu Webhook",
  "avatar": "data:image/png;base64,<base64>"
}
```

**Resposta:**

```json
{
  "id": "webhook_id",
  "type": 1,
  "guild_id": "guild_id",
  "channel_id": "channel_id",
  "name": "Meu Webhook",
  "avatar": null,
  "token": "token_do_webhook",
  "url": "https://discord.com/api/webhooks/{id}/{token}"
}
```

### Listar webhooks de um canal

```http
GET /channels/{channel_id}/webhooks
Authorization: <token>
```

### Listar webhooks de um servidor

```http
GET /guilds/{guild_id}/webhooks
Authorization: <token>
```

### Executar webhook (enviar mensagem)

```http
POST /webhooks/{webhook_id}/{webhook_token}
Content-Type: application/json

{
  "content": "Mensagem do webhook!",
  "username": "Nome customizado",
  "avatar_url": "https://exemplo.com/avatar.png",
  "tts": false,
  "embeds": [
    {
      "title": "Título do Embed",
      "description": "Descrição",
      "color": 5763719
    }
  ]
}
```

> Adicione `?wait=true` para receber o objeto da mensagem criada na resposta.

**Limites dos embeds via webhook:**

| Campo          | Limite     |
|----------------|------------|
| `title`        | 256 chars  |
| `description`  | 4096 chars |
| `field.name`   | 256 chars  |
| `field.value`  | 1024 chars |
| `footer.text`  | 2048 chars |
| `author.name`  | 256 chars  |
| Total embed    | 6000 chars |
| Embeds por msg | até 10     |

### Editar mensagem do webhook

```http
PATCH /webhooks/{webhook_id}/{webhook_token}/messages/{message_id}
Content-Type: application/json

{
  "content": "Mensagem editada"
}
```

### Deletar mensagem do webhook

```http
DELETE /webhooks/{webhook_id}/{webhook_token}/messages/{message_id}
```

### Editar o webhook

```http
PATCH /webhooks/{webhook_id}/{webhook_token}
Content-Type: application/json

{
  "name": "Novo Nome",
  "channel_id": "novo_channel_id"
}
```

### Deletar o webhook

```http
DELETE /webhooks/{webhook_id}/{webhook_token}
```

---

## 11. Busca de Mensagens

O Discord possui um endpoint de busca que **não está na API oficial de bots** — é exclusivo do client de usuário.

### Buscar mensagens em um canal

```http
GET /channels/{channel_id}/messages/search?q=termo&limit=25
Authorization: <token>
```

### Buscar mensagens em um servidor inteiro

```http
GET /guilds/{guild_id}/messages/search?q=termo&limit=25
Authorization: <token>
```

**Parâmetros disponíveis:**

| Parâmetro    | Tipo      | Descrição                                                         |
|--------------|-----------|-------------------------------------------------------------------|
| `q`          | string    | Termo de busca (obrigatório)                                      |
| `limit`      | integer   | 1–25, default 25                                                  |
| `offset`     | integer   | Paginação por offset                                              |
| `author_id`  | snowflake | Filtrar por autor                                                 |
| `mentions`   | snowflake | Mensagens que mencionam este usuário                              |
| `has`        | string    | `link`, `embed`, `file`, `video`, `image`, `sound`, `sticker`    |
| `channel_id` | snowflake | Restringir a um canal específico (em guilds)                      |
| `min_id`     | snowflake | Snowflake mínimo (data mais antiga)                               |
| `max_id`     | snowflake | Snowflake máximo (data mais recente)                              |
| `sort_by`    | string    | `timestamp` ou `relevance`                                        |
| `sort_order` | string    | `asc` ou `desc`                                                   |
| `pinned`     | boolean   | Apenas mensagens fixadas                                          |

**Resposta:**

```json
{
  "messages": [
    [ { "id": "...", "content": "Mensagem encontrada", "..." : "..." } ]
  ],
  "total_results": 142,
  "analytics_id": "abc123..."
}
```

> Cada item em `messages` é um array — o primeiro elemento é a mensagem encontrada, e os demais são mensagens vizinhas fornecidas como contexto.

---

## 12. Convites (Invites)

### Obter informações de um convite

```http
GET /invites/{invite_code}?with_counts=true&with_expiration=true
```

> Este endpoint **não precisa de autenticação** — é público.

**Resposta:**

```json
{
  "code": "discord",
  "type": 0,
  "expires_at": null,
  "guild": {
    "id": "...",
    "name": "Discord",
    "splash": null,
    "banner": null,
    "icon": "...",
    "description": "...",
    "features": [],
    "verification_level": 0,
    "vanity_url_code": "discord"
  },
  "channel": {
    "id": "...",
    "type": 0,
    "name": "general"
  },
  "inviter": { "id": "...", "username": "...", "avatar": "..." },
  "approximate_member_count": 123456,
  "approximate_presence_count": 45678
}
```

### Criar convite em um canal

```http
POST /channels/{channel_id}/invites
Authorization: <token>
Content-Type: application/json

{
  "max_age": 86400,
  "max_uses": 10,
  "temporary": false,
  "unique": true
}
```

**Parâmetros:**

| Campo       | Tipo    | Descrição                                    |
|-------------|---------|----------------------------------------------|
| `max_age`   | integer | Duração em segundos (0 = sem expiração)      |
| `max_uses`  | integer | Usos máximos (0 = ilimitado)                 |
| `temporary` | boolean | Membro é kickado ao sair da voz              |
| `unique`    | boolean | Gerar código único mesmo para configs iguais |

### Deletar convite

```http
DELETE /invites/{invite_code}
Authorization: <token>
```

### Listar convites de um canal

```http
GET /channels/{channel_id}/invites
Authorization: <token>
```

---

## 13. Formatação de Texto (Markdown)

O Discord suporta um subset do Markdown com extensões próprias.

### Formatação básica

| Sintaxe                | Resultado             |
|------------------------|-----------------------|
| `**texto**`            | **negrito**           |
| `*texto*` ou `_texto_` | *itálico*             |
| `***texto***`          | ***negrito+itálico*** |
| `__texto__`            | sublinhado            |
| `~~texto~~`            | ~~tachado~~           |
| `` `código` ``         | `código inline`       |
| ` ```código``` `       | bloco de código       |
| `> texto`              | citação               |
| `>>> texto`            | citação em bloco      |
| `# Título`             | cabeçalho H1          |
| `## Título`            | cabeçalho H2          |
| `### Título`           | cabeçalho H3          |
| `- item` ou `* item`   | lista                 |
| `1. item`              | lista numerada        |
| `\|\|spoiler\|\|`      | texto de spoiler      |

### Mentions e IDs

| Sintaxe          | Tipo                        |
|------------------|-----------------------------|
| `<@USER_ID>`     | Mencionar usuário           |
| `<@!USER_ID>`    | Mencionar membro (com nick) |
| `<#CHANNEL_ID>`  | Mencionar canal             |
| `<@&ROLE_ID>`    | Mencionar cargo             |
| `@everyone`      | Mencionar todos             |
| `@here`          | Mencionar membros online    |

### Emojis

| Sintaxe          | Tipo                       |
|------------------|----------------------------|
| `:emoji_name:`   | Emoji Unicode por nome     |
| `<:nome:ID>`     | Emoji customizado estático |
| `<a:nome:ID>`    | Emoji customizado animado  |

### Timestamps

O Discord suporta timestamps dinâmicos que se adaptam ao fuso horário do usuário:

```
<t:UNIX_TIMESTAMP>      → Data/hora padrão
<t:UNIX_TIMESTAMP:d>    → Data curta:  20/01/2025
<t:UNIX_TIMESTAMP:D>    → Data longa:  20 de janeiro de 2025
<t:UNIX_TIMESTAMP:t>    → Hora curta:  16:20
<t:UNIX_TIMESTAMP:T>    → Hora longa:  16:20:30
<t:UNIX_TIMESTAMP:f>    → Data + hora: 20 de janeiro de 2025 16:20
<t:UNIX_TIMESTAMP:F>    → Completo:    segunda-feira, 20 de janeiro de 2025 16:20
<t:UNIX_TIMESTAMP:R>    → Relativo:    há 2 dias / em 3 horas
```

**Exemplo:**

```
Reunião: <t:1737993600:F> (<t:1737993600:R>)
→ Reunião: segunda-feira, 27 de janeiro de 2025 18:00 (em 2 dias)
```
