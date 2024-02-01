# Relay HTTP REST API

- Author: [Sébastien Helleu](https://github.com/flashcode)
- License: CC BY-NC-SA 4.0
- Created on: 2023-12-05
- Last updated: 2024-02-01
- Issues:
  - [#2066](https://github.com/weechat/weechat/issues/2066): new relay "api": HTTP REST API
  - [#1549](https://github.com/weechat/weechat/issues/1549): add support of websocket extension "permessage-deflate"
- Status: draft
- Target WeeChat version: 4.3.0

## Context

WeeChat allows external clients to connect via two protocols in relay plugin:

- `irc`: any IRC client can connect (including WeeChat itself)
- `weechat`: only specific clients can connect (this does NOT include WeeChat)

The `weechat` protocol uses a custom binary protocol that is complicated to
implement in a client and exposes internal structures with pointers, which is unsafe.

## Goals

Purpose of this specification is to add a third relay protocol called `api`, with the following goals:

- easy client implementation:
  - HTTP REST API exposed by WeeChat, can be used from command line (`curl`)
  - JSON format for input/output
  - automatic compression of responses (deflate, gzip, zstd and permessage-deflate for websocket)
- data synchronization:
  - optional JWT token to make multiple requests with a single authentication
  - real-time sync with websocket or polling with HTTP requests
- no internal structures are exposed:
  - color codes in messages can be converted to ANSI colors, kept as-is or stripped
  - no use of pointers
  - no use of complex structures like hdata
- WeeChat itself is able to connect to another WeeChat using this protocol

## Out of scope

Existing relay protocols `irc` and `weechat` are unchanged.

Protocol `weechat` will benefit of websocket extension `permessage-deflate` (compression of messages exchanged with the client, in both directions). \
This is transparent for existing clients: if they do not support this extension, WeeChat will still send uncompressed messages to the client.

## Changes

In the examples below, the JSON output is formatted for readability, but the WeeChat response is sent without any indentation, for example:

```json
{"weechat_version":"4.2.0-dev","weechat_version_git":"v4.1.0-143-g0b1cda1c4",...}
```

### New relay protocol

A new relay protocol called `api` is added.

Example:

```text
/relay add tls.api 9000
```

This protocol allows communication using a HTTP REST API exposed by WeeChat/relay. \
All resources start with `/api/`, followed by the resource name.

The following methods and paths are available:

- [`POST /api/handshake`](#resource-handshake): handshake with WeeChat (no authentication required)
- [`GET /api/version`](#resource-version): get the WeeChat and relay API versions
- [`GET /api/buffers`](#resource-buffers): get buffers, lines, nicks
- [`POST /api/input`](#resource-input): send command or text to a buffer
- [`POST /api/ping`](#resource-ping): send a "ping" request
- [`POST /api/sync`](#resource-sync): synchronize with WeeChat

The following HTTP response codes can be sent back to the client:

- `200 OK`: response OK with a body (JSON)
- `204 No Content`: response OK with empty body
- `400 Bad Request`: invalid body received
- `401 Unauthorized`: missing or invalid credentials
- `403 Forbidden`: forbidden (no permission)
- `404 Not Found`: resource not found
- `500 Internal Server Error`: internal server error
- `503 Service Unavailable`: service unavailable

When connected via websocket, an extra response code is sent when WeeChat pushes data to the client on events:

- `0 Event`: event pushed to the websocket client, if synchronization is enabled with [Sync resource](#resource-sync)

### API schema

The description of API is available as an OpenAPI document, auto-generated when WeeChat is built (relay plugin must be enabled in the build).

### API versioning

The API is versioned using a "practical" [Semantic Versioning](https://semver.org), like WeeChat, on three digits `X.Y.Z`, where:

- `X` is the major version
- `Y` is the minor version
- `Z` is the patch version.

Example: version `2.0.0` brings breaking changes vs version `1.2.3`.

The API version is returned by the [Version resource](#resource-version).

### Date format

The date format is [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601), using UTC timezone (so this can differ from the dates displayed in WeeChat itself, which is using the local timezone).

The dates are returned with maximum precision: up to microseconds if available (if not, it may include milliseconds, or nothing after the seconds).

Examples:

```text
2023-12-05T19:46:03.847625Z
2023-12-05T19:46:03.847Z
2023-12-05T19:46:03Z
```

### Authentication

The password must be sent in the header `Authorization` with `Basic` authentication schema.

The password can be sent as plain text or hashed, with one of these formats for user and password:

- `plain:<password>`
- `hash:sha256:<salt>:<hash>`
- `hash:sha512:<salt>:<hash>`
- `hash:pbkdf2+sha256:<salt>:<iterations>:<hash>`
- `hash:pbkdf2+sha512:<salt>:<iterations>:<hash>`

Where:

- `<password>` is the password as plain text
- `<salt>`: salt: the current timestamp as integer (number of seconds since the Unix Epoch); it is used to prevent replay attacks
- `<iterations>`: number of iterations (for PBKDF2 algorithm only)
- `<hash>`: the hashed salt + password (hexadecimal)

A new option `relay.network.time_window` is added in WeeChat to set the max number of seconds allowed before and after the received time (when password is sent hashed). Default value is 5.

Example:

- current timestamp is `1706431066`
- password is `secret_password`
- hash algorithm is `sha256`
- result hash is the SHA256 of string `1706431066secret_password` which is as hexadecimal: `dfa1db3f6bb6445d18d9ec7427c10f6421274e3a4751e6c1ffc7dd28c94eadf6`
- the `Authorization` header is the base64 encoded string `hash:sha256:1706431066:dfa1db3f6bb6445d18d9ec7427c10f6421274e3a4751e6c1ffc7dd28c94eadf6`: `aGFzaDpzaGEyNTY6MTcwNjQzMTA2NjpkZmExZGIzZjZiYjY0NDVkMThkOWVjNzQyN2MxMGY2NDIxMjc0ZTNhNDc1MWU2YzFmZmM3ZGQyOGM5NGVhZGY2`.

The header `Authorization` is allowed in the first websocket request (see [Handshake](#handshake)) or any HTTP request when websocket is not used and when a JWT token is not sent.

Request example with plain password:

```bash
curl -L -u plain:secret_password' 'https://localhost:9000/api/version'
```

Request example with hashed password (SHA256):

```bash
curl -L -u 'hash:sha256:1706431066:dfa1db3f6bb6445d18d9ec7427c10f6421274e3a4751e6c1ffc7dd28c94eadf6' \
  'https://localhost:9000/api/version'
```

If TOTP is enabled on WeeChat/relay side (option `relay.network.totp_secret` is set), you must send the TOTP value in the `x-weechat-totp` header like this:

```bash
curl -L -u 'hash:sha256:1706431066:dfa1db3f6bb6445d18d9ec7427c10f6421274e3a4751e6c1ffc7dd28c94eadf6' \
  -H "x-weechat-totp: 123456" 'https://localhost:9000/api/version'
```

In case of error, a response `401 Unauthorized` is returned with a field `error` in JSON that describes the error.

Response: missing password:

```text
HTTP/1.1 401 Unauthorized
```

```json
{
    "error": "Missing password"
}
```

Response: wrong password:

```text
HTTP/1.1 401 Unauthorized
```

```json
{
    "error": "Invalid password"
}
```

Response: invalid hash algorithm:

```text
HTTP/1.1 401 Unauthorized
```

```json
{
    "error": "Invalid hash algorithm (not found or not supported)"
}
```

Response: invalid salt:

```text
HTTP/1.1 401 Unauthorized
```

```json
{
    "error": "Invalid salt"
}
```

Response: invalid number of iterations:

```text
HTTP/1.1 401 Unauthorized
```

```json
{
    "error": "Invalid number of iterations"
}
```

Response: missing TOTP:

```text
HTTP/1.1 401 Unauthorized
```

```json
{
    "error": "Missing TOTP"
}
```

Response: wrong TOTP:

```text
HTTP/1.1 401 Unauthorized
```

```json
{
    "error": "Invalid TOTP"
}
```

### Compression

Compression of response body is automatic and based on header `Accept-Encoding` sent by the client.

Supported compression formats:

- `deflate` (zlib)
- `gzip`
- `zstd`

Request example:

```bash
curl -L -u 'plain:secret_password' -H "Accept-Encoding: gzip" 'https://localhost:9000/api/version'
```

Response:

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Encoding: gzip
Content-Length: 77
```

```text
[77 bytes data]
```

Note: with websocket connection, the extension `permessage-deflate` allows to compress messages with zlib.

### Resource: handshake

Perform an handshake between the client and WeeChat.

This resource does not require authentication.

Endpoint:

```http
POST /api/handshake
```

Body parameters:

- `password_hash_algo` (array of strings, optional): list of hash algorithms supported by the client, each string can be:
  - `plain`: plain-text password (no hash)
  - `sha256`: hash SHA256
  - `sha512`: hash SHA512
  - `pbkdf2+sha256`: hash PBKDF2 with SHA256
  - `pbkdf2+sha512`: hash PBKDF2 with SHA512

The response has the following fields:

- `password_hash_algo` (string): the hash algorithm to use (`null` if no algorithm is compatible)
- `password_hash_iterations` (integer): the number of iterations to use if hash PBKDF2 is used
- `totp` (boolean): `true` if TOTP is enabled in WeeChat (then the client must sent TOTP in specific header), `false` otherwise

Request example:

```bash
curl -L -X POST -d '{"password_hash_algo": ["plain", "sha256", "sha512"]}' 'https://localhost:9000/api/handshake'
```

Response:

```text
HTTP/1.1 200 OK
```

```json
{
    "password_hash_algo": "sha512",
    "password_hash_iterations": 100000,
    "totp": false
}
```

### Resource: version

Return the WeeChat and relay API versions.

Endpoint:

```http
GET /api/version
```

Request example:

```bash
curl -L -u 'plain:secret_password' 'https://localhost:9000/api/version'
```

Response:

```text
HTTP/1.1 200 OK
```

```json
{
    "weechat_version": "4.2.0-dev",
    "weechat_version_git": "v4.1.0-143-g0b1cda1c4",
    "weechat_version_number": 67239936,
    "relay_api_version": "0.0.1",
    "relay_api_version_number": 1
}
```

### Resource: buffers

Return buffers, lines and nicks.

Endpoints:

```http
GET /api/buffers
GET /api/buffers/{buffer_name}
```

Path parameters:

- `buffer_name` (string, optional): buffer name

Query parameters:

- `lines` (integer, optional, default: `0`): number of lines in buffer to return:
  - negative number: return N lines from the end of buffer (newest lines)
  - `0`: do not return any line
  - positive number: return N lines from the beginning of buffer (oldest lines)
- `nicks` (boolean, optional, default: `false`): return nicks in buffer
- `colors` (string, optional, default: `ansi`): how to return strings with color codes:
  - `ansi`: return ANSI color codes
  - `weechat`: return WeeChat internal color codes
  - `strip`: strip colors

Request example: get all buffers without lines:

```bash
curl -L -u 'plain:secret_password' 'https://localhost:9000/api/buffers'
```

Response:

```text
HTTP/1.1 200 OK
```

```json
[
    {
        "name": "core.weechat",
        "short_name": "weechat",
        "number": 1,
        "type": "formatted",
        "title": "WeeChat 4.2.0-dev (C) 2003-2023 - https://weechat.org/",
        "local_variables": {
            "plugin": "core",
            "name": "weechat"
        }
    },
    {
        "name": "irc.server.libera",
        "short_name": "libera",
        "number": 2,
        "type": "formatted",
        "title": "IRC: irc.libera.chat/6697 (2001:4b7a:a008::6667)",
        "local_variables": {
            "plugin": "irc",
            "name": "server.libera",
            "type": "server",
            "server": "libera",
            "channel": "libera",
            "charset_modifier": "irc.libera",
            "nick": "alice",
            "tls_version": "TLS1.3",
            "host": "~alice@example.com"
        }
    },
    {
        "name": "irc.libera.#weechat",
        "short_name": "#weechat",
        "number": 3,
        "type": "formatted",
        "title": "Welcome to the WeeChat official support channel",
        "local_variables": {
            "plugin": "irc",
            "name": "libera.#weechat",
            "type": "channel",
            "server": "libera",
            "channel": "#weechat",
            "nick": "alice",
            "host": "~alice@example.com"
        }
    }
]
```

Request example: get WeeChat core buffer with only last line:

```bash
curl -L -u 'plain:secret_password' \
  'https://localhost:9000/api/buffers/core.weechat?lines=-1&colors=strip'
```

Response:

```text
HTTP/1.1 200 OK
```

```json
{
    "name": "core.weechat",
    "short_name": "weechat",
    "number": 1,
    "type": "formatted",
    "title": "WeeChat 4.2.0-dev (C) 2003-2023 - https://weechat.org/",
    "local_variables": {
        "plugin": "core",
        "name": "weechat"
    },
    "lines": [
        {
            "id": 10,
            "y": -1,
            "date": "2023-12-24T08:17:20.786538Z",
            "date_printed": "2023-12-24T08:17:20.786538Z",
            "highlight": false,
            "prefix": "",
            "message": "Plugins loaded: alias, buflist, charset, exec, fifo, fset, guile, irc, javascript, logger, lua, perl, php, python, relay, ruby, script, spell, tcl, trigger, typing, xfer",
            "tags": []
        }
    ]
}
```

#### Sub-resource: buffers / lines

Return lines in a buffer.

Endpoints:

```http
GET /api/buffers/{buffer_name}/lines
GET /api/buffers/{buffer_name}/lines/{line_id}
```

Path parameters:

- `buffer_name` (string, **required**): buffer name
- `line_id` (integer, optional): return a single line with this identifier

Query parameters:

- `lines` (integer, optional, default: `-100`): number of lines in buffer to return:
  - negative number: return N lines from the end of buffer (newest lines)
  - `0`: do not return any line (allowed but doesn't make sense with this resource)
  - positive number: return N lines from the beginning of buffer (oldest lines)
- `colors` (string, optional, default: `ansi`): how to return strings with color codes:
  - `ansi`: return ANSI color codes
  - `weechat`: return WeeChat internal color codes
  - `strip`: strip colors

Request example: get last 1000 lines of a buffer:

```bash
curl -L -u 'plain:secret_password' \
  'https://localhost:9000/api/buffers/irc.libera.%23weechat/lines?lines=-1000&colors=strip'
```

Response:

```text
HTTP/1.1 200 OK
```

```json
[
    {
        "id": 0,
        "y": -1,
        "date": "2023-12-05T19:46:03.847625Z",
        "date_printed": "2023-12-05T19:46:03.847625Z",
        "highlight": false,
        "prefix": "-->",
        "message": "alice (~alice@example.com) has joined #test",
        "tags": [
            "irc_join",
            "irc_tag_account=alice",
            "irc_tag_time=2023-12-05T19:46:03.847Z",
            "nick_alice",
            "host_~alice@example.com",
            "log4"
        ]
    },
    {
        "id": 1,
        "y": -1,
        "date": "2023-12-05T19:46:03.986543Z",
        "date_printed": "2023-12-05T19:46:03.986543Z",
        "highlight": false,
        "prefix": "--",
        "message": "Mode #test [+Cnst] by zirconium.libera.chat",
        "tags": [
            "irc_mode",
            "irc_tag_time=2023-12-05T19:46:03.986Z",
            "nick_zirconium.libera.chat",
            "log3"
        ]
    },
    {
        "id": 2,
        "y": -1,
        "date": "2023-12-05T19:46:04.287546Z",
        "date_printed": "2023-12-05T19:46:04.287546Z",
        "highlight": false,
        "prefix": "--",
        "message": "Channel #test: 1 nick (1 op, 0 voiced, 0 regular)",
        "tags": [
            "irc_366",
            "irc_numeric",
            "irc_tag_time=2023-12-05T19:46:04.287Z",
            "nick_zirconium.libera.chat",
            "log3"
        ]
    }
]
```

#### Sub-resource: buffers / nicks

Return nicks in a buffer.

Endpoint:

```http
GET /api/buffers/{buffer_name}/nicks
```

Path parameters:

- `buffer_name` (string, **required**): buffer name

Request example: get nicks of a buffer:

```bash
curl -L -u 'plain:secret_password' \
  'https://localhost:9000/api/buffers/irc.libera.%23weechat/nicks'
```

Response:

```text
HTTP/1.1 200 OK
```

```json
{
    "name": "root",
    "color": "",
    "groups": [
        {
            "name": "000|o",
            "color": "green",
            "groups": [],
            "nicks": [
                {
                    "prefix": "@",
                    "prefix_color": "lightgreen",
                    "name": "alice",
                    "color": "default"
                }
            ]
        },
        {
            "name": "001|h",
            "color": "green",
            "groups": [],
            "nicks": []
        },
        {
            "name": "002|v",
            "color": "green",
            "groups": [],
            "nicks": []
        },
        {
            "name": "999|...",
            "color": "green",
            "groups": [],
            "nicks": []
        }
    ],
    "nicks": []
}
```

### Resource: input

Send command or text to a buffer.

Endpoint:

```http
POST /api/input
```

Body parameters:

- `buffer` (string, optional, default: `core.weechat`): buffer name where the command is executed
- `command` (string, **required**): command or text to send to the buffer

Request example: say "hello!" on channel #weechat:

```bash
curl -L -u 'plain:secret_password' -X POST \
  -d '{"buffer": "irc.libera.#weechat", "command": "hello!"}' \
  'https://localhost:9000/api/input'
```

Response:

```text
HTTP/1.1 204 No content
```

Request example: part and close channel #weechat (command executed on WeeChat core buffer):

```bash
curl -L -u 'plain:secret_password' -X POST \
  -d '{"command": "/buffer close irc.libera.#weechat"}' \
  'https://localhost:9000/api/input'
```

Response:

```text
HTTP/1.1 204 No content
```

### Resource: ping

Send a "ping" request.

Endpoint:

```http
POST /api/ping
```

Body parameters:

- `data` (string, optional): string that is sent back in the response

Request example: no body:

```bash
curl -L -u 'plain:secret_password' -X POST 'https://localhost:9000/api/ping'
```

Response:

```text
HTTP/1.1 204 No content
```

Request example: with data:

```bash
curl -L -u 'plain:secret_password' -X POST \
  -d '{"data": "1702835741"}' \
  'https://localhost:9000/api/ping'
```

Response:

```text
HTTP/1.1 200 OK
```

```json
{
    "data": "1702835741"
}
```

### Resource: sync

Start/stop synchronization of data with WeeChat.

This resource can be used only when the client is connected with a websocket, as WeeChat will push messages to the client at any time using the websocket.

If this resource is used without a websocket connection, an error 403 (Forbidden) is returned.

Endpoint:

```http
POST /api/sync
```

Body parameters:

- `sync` (boolean, optional, default: `true`): `true` to enable synchronization with WeeChat
- `nicks` (boolean, optional, default: `true`): `true` to receive nick updates in buffers (used only if `sync` is `true`)
- `colors` (string, optional, default: `ansi`): how to return strings with color codes (used only if `sync` is `true`):
  - `ansi`: return ANSI color codes
  - `weechat`: return WeeChat internal color codes
  - `strip`: strip colors

Request example with websocket:

```json
{
    "request": "POST /api/sync",
    "body": {
        "nicks": false
    }
}
```

Response:

```json
{
    "code": 204,
    "message": "No Content"
}
```

Request example without websocket (error):

```bash
curl -L -u 'plain:secret_password' -X POST \
  -d '{"nicks": false}' \
  'https://localhost:9000/api/sync'
```

Response:

```text
HTTP/1.1 403 Forbidden
```

```json
{
    "error": "Sync resource is available only with a websocket connection"
}
```

### Websocket

Websocket is used to make a persistent connection between the client and WeeChat and receive events in real-time if synchronization is enabled with [Sync resource](#resource-sync).

Authentication must be done only one time when a websocket is used (see [Authentication](#authentication).

#### Handshake

To establish the connection, a handshake is performed. The client's handshake must be done on endpoint `/api` and looks like:

```http
GET /api HTTP/1.1
Host: localhost:9000
Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36
Upgrade: websocket
Origin: https://example.com
Sec-WebSocket-Version: 13
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9,fr;q=0.8
Sec-WebSocket-Key: 2XE8VAJktqi3Tpw5QnfxVQ==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```

WeeChat sends its handshake to confirm that websocket protocol is properly supported (and authentication was successful):

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: PaY9vRflWeOKuD0/F7e5gD9At9U=
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```

Note: the `Sec-WebSocket-Accept` value returned is the SHA-1 hash of the value received, concatenated to the GUID "258EAFA5-E914-47DA-95CA-C5AB0DC85B11" (the SHA-1 is encoded in base64). \
In the example above, the SHA-1 of "2XE8VAJktqi3Tpw5QnfxVQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11" is "PaY9vRflWeOKuD0/F7e5gD9At9U=" (in base64).

The support of `permessage-deflate` is added, that's why WeeChat includes it in the handshake reply.

#### Frames

When the client is connected via a websocket:

- requests and responses are in JSON, inside websocket frames
- if synchronization is enabled with [Sync resource](#resource-sync), WeeChat can send JSON frames at any time to the client.

Requests to WeeChat are made with an object containing these fields:

- `request` (string): the HTTP method and path (example: `GET /api/version`)
- `body` (object or array): the body (optional, for `POST` and `PUT` methods)

Responses to client are made with an object containing these fields:

- `code` (integer): HTTP response code (example: `200`)
- `message` (string): message for the code (example: `OK`)
- `body` (object or array): the body returned

Request example: get version:

```json
{
    "request": "GET /api/version"
}
```

Response:

```json
{
    "code": 200,
    "message": "OK",
    "body": {
        "weechat_version": "4.2.0-dev",
        "weechat_version_git": "v4.1.0-143-g0b1cda1c4",
        "weechat_version_number": 67239936,
        "relay_api_version": "0.0.1",
        "relay_api_version_number": 1
    }
}
```

Request example: say "hello!" on channel #weechat:

```json
{
    "request": "POST /api/input",
    "body": {
        "buffer": "irc.libera.#weechat",
        "command": "hello!"
    }
}
```

Response:

```json
{
    "code": 204,
    "message": "No Content"
}
```

WeeChat pushes data to the client at any time on some events: when lines are displayed, buffers added/removed/changed, nicks added/removed/changed, etc.

The JSON sent has `code` set to `0`, `message` set to `Event` and an extra object `event` with the following data:

- `name` (string): the event name (name of signal or hsignal)
- `type` (string): the object type returned in body (empty string if no body is returned)
- `context` (string): the context for the object returned, set only for sub-objects, empty string in other cases (for example it contains the buffer name when line or nick is sent)

The following events are sent to the client, according to synchronization options:

Event                     | Type of body   | Body context | Description of data sent
------------------------- | ---------------|--------------|-------------------------
`buffer_opened`           | `buffer`       | N/A          | buffer with all lines
`buffer_type_changed`     | `buffer`       | N/A          | buffer
`buffer_moved`            | `buffer`       | N/A          | buffer
`buffer_merged`           | `buffer`       | N/A          | buffer
`buffer_unmerged`         | `buffer`       | N/A          | buffer
`buffer_hidden`           | `buffer`       | N/A          | buffer
`buffer_unhidden`         | `buffer`       | N/A          | buffer
`buffer_renamed`          | `buffer`       | N/A          | buffer
`buffer_title_changed`    | `buffer`       | N/A          | buffer
`buffer_localvar_xxx`     | `buffer`       | N/A          | buffer
`buffer_cleared`          | `buffer`       | N/A          | buffer
`buffer_closing`          | `buffer`       | N/A          | buffer
`buffer_line_added`       | `line`         | buffer name  | buffer line
`upgrade`                 | (empty string) | N/A          | (no body)
`upgrade_ended`           | (empty string) | N/A          | (no body)
`nicklist_group_added`    | `nick_group`   | buffer name  | nick group
`nicklist_group_changed`  | `nick_group`   | buffer name  | nick group
`nicklist_group_removing` | `nick_group`   | buffer name  | nick group
`nicklist_nick_added`     | `nick`         | buffer name  | nick
`nicklist_nick_removing`  | `nick`         | buffer name  | nick
`nicklist_nick_changed`   | `nick`         | buffer name  | nick

Example: new buffer: channel `#weechat` has been joined:

```json
{
    "code": 0,
    "message": "Event",
    "event": {
        "name": "buffer_opened",
        "type": "buffer",
        "context": ""
    },
    "body": {
        "name": "irc.libera.#test",
        "short_name": "",
        "number": 4,
        "type": "formatted",
        "title": "",
        "local_variables": {
            "plugin": "irc",
            "name": "libera.#test",
            "type": "channel",
            "nick": "alice",
            "host": "~alice@example.com",
            "server": "libera",
            "channel": "#test"
        },
        "lines": []
    }
}
```

Example: new line displayed on channel `#weechat`:

```json
{
    "code": 0,
    "message": "Event",
    "event": {
        "name": "buffer_line_added",
        "type": "line",
        "context": "irc.libera.#weechat"
    },
    "body": {
        "id": 5,
        "index": -1,
        "date": "2024-01-07T08:54:00.179483Z",
        "date_printed": "2024-01-07T08:54:00.179483Z",
        "highlight": false,
        "prefix": "alice",
        "message": "hello!",
        "tags": [
            "irc_privmsg",
            "self_msg",
            "notify_none",
            "no_highlight",
            "prefix_nick_white",
            "nick_alice",
            "log1"
        ]
    }
}
```

Example: nick `bob` added as operator in channel `#weechat`:

```json
{
    "code": 0,
    "message": "Event",
    "event": {
        "name": "nicklist_nick_added",
        "type": "nick",
        "context": "irc.libera.#weechat"
    },
    "body": {
        "prefix": "@",
        "prefix_color": "lightgreen",
        "name": "bob",
        "color": "default"
    }
}
```

Example: WeeChat is upgrading:

```json
{
    "code": 0,
    "message": "Event",
    "event": {
        "name": "upgrade",
        "type": "",
        "context": ""
    }
}
```

Example: upgrade has been done:

```json
{
    "code": 0,
    "message": "Event",
    "event": {
        "name": "upgrade_ended",
        "type": "",
        "context": ""
    }
}
```

## Planning

The changes must be implemented in this order:

1. Add "api" protocol with JSON support and plain-text password authentication
2. Add support of websocket extension "permessage-deflate"
3. Add support of hashed passwords (SHA and PBKDF2)
4. Add "handshake" resource

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-005-relay-http-rest-api.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-005-relay-http-rest-api.md)
