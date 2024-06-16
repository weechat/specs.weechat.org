# Relay HTTP REST API

- Author: [Sébastien Helleu](https://github.com/flashcode)
- License: CC BY-NC-SA 4.0
- Created on: 2023-12-05
- Last updated: 2024-06-16
- Issues:
  - [#2066](https://github.com/weechat/weechat/issues/2066): new relay "api": HTTP REST API
  - [#1549](https://github.com/weechat/weechat/issues/1549): add support of websocket extension "permessage-deflate"
  - [#369](https://github.com/weechat/weechat/issues/369): add connection to WeeChat remote relay
- Status: implemented
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
- data synchronization: real-time sync with websocket or polling with HTTP requests
- no internal structures are exposed:
  - color codes in messages can be converted to ANSI colors, kept as-is or stripped
  - no use of pointers
  - no use of complex structures like hdata
- WeeChat itself is able to connect to another WeeChat using this protocol

## Out of scope

Existing relay protocols `irc` and `weechat` are unchanged.

~~Protocol `weechat` will benefit of websocket extension `permessage-deflate` (compression of messages exchanged with the client, in both directions).~~ \
~~This is transparent for existing clients: if they do not support this extension, WeeChat will still send uncompressed messages to the client.~~\
**Update on 2024-06-06**: as the extension `permessage-deflate` is causing troubles with some browsers like Safari (see this glowing-bear [issue](https://github.com/glowing-bear/glowing-bear/issues/1280)), it is now enabled only with "api" relay (WeeChat ≥ 4.3.2).

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

- [`OPTIONS /api/xxx`](#preflight-request): preflight request (CORS) (no authentication required)
- [`POST /api/handshake`](#resource-handshake): handshake with WeeChat (no authentication required)
- [`GET /api/version`](#resource-version): get the WeeChat and relay API versions
- [`GET /api/buffers`](#resource-buffers): get buffers, lines, nicks
- [`GET /api/hotlist`](#resource-hotlist): get hotlist
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
- `hash:sha256:<timestamp>:<hash>`
- `hash:sha512:<timestamp>:<hash>`
- `hash:pbkdf2+sha256:<timestamp>:<iterations>:<hash>`
- `hash:pbkdf2+sha512:<timestamp>:<iterations>:<hash>`

Where:

- `<password>` is the password as plain text
- `<timestamp>` is the current timestamp as integer (number of seconds since the Unix Epoch); it is used to prevent replay attacks
- `<iterations>` is the number of iterations (for PBKDF2 algorithm only)
- `<hash>` is the hashed timestamp + password (hexadecimal)

A new option `relay.network.time_window` is added in WeeChat to set the max number of seconds allowed before and after the received time (when password is sent hashed). Default value is 5.

Example:

- current timestamp is `1706431066`
- password is `secret_password`
- hash algorithm is `sha256`
- result hash is the SHA256 of string `1706431066secret_password` which is as hexadecimal: `dfa1db3f6bb6445d18d9ec7427c10f6421274e3a4751e6c1ffc7dd28c94eadf6`
- the `Authorization` header is the base64 encoded string `hash:sha256:1706431066:dfa1db3f6bb6445d18d9ec7427c10f6421274e3a4751e6c1ffc7dd28c94eadf6`: `aGFzaDpzaGEyNTY6MTcwNjQzMTA2NjpkZmExZGIzZjZiYjY0NDVkMThkOWVjNzQyN2MxMGY2NDIxMjc0ZTNhNDc1MWU2YzFmZmM3ZGQyOGM5NGVhZGY2`.

The header `Authorization` is allowed in the first websocket request (see [Handshake](#handshake)) or any HTTP request when websocket is not used.

Request example with plain password:

```bash
curl -L -u 'plain:secret_password' 'https://localhost:9000/api/version'
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

Response: invalid timestamp:

```text
HTTP/1.1 401 Unauthorized
```

```json
{
    "error": "Invalid timestamp"
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

### Preflight request

The preflight request with HTTP method `OPTIONS` is used by browsers to check that the server (WeeChat) will permit the actual request.

Request example:

```http
OPTIONS /api/version HTTP/1.1
Host: localhost:9000
Connection: keep-alive
Accept: */*
Access-Control-Request-Method: GET
Access-Control-Request-Headers: authorization
Origin: http://localhost
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123.0.0.0 Safari/537.36
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-site
Sec-Fetch-Dest: empty
Referer: http://localhost/
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: en-US,en;q=0.9,fr;q=0.8
```

Response:

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: origin, content-type, accept, authorization
Access-Control-Allow-Origin: *
Content-Type: application/json; charset=utf-8
Content-Length: 0
```

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
- `totp` (boolean): `true` if TOTP is enabled in WeeChat (then the client must send TOTP in specific header), `false` otherwise

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
GET /api/buffers/{buffer_id}
GET /api/buffers/{buffer_name}
```

Path parameters:

- `buffer_id` (integer, optional): buffer unique identifier (not to be confused with the buffer number, which is different)
- `buffer_name` (string, optional): buffer name

Query parameters:

- `lines` (integer, optional, default: `0`): number of lines to return in buffers with formatted content:
  - negative number: return N lines from the end of buffer (newest lines)
  - `0`: do not return any line
  - positive number: return N lines from the beginning of buffer (oldest lines)
- `lines_free` (integer, optional, default: `0` if `lines` is `0`, otherwise all lines): number of lines to return in buffers with free content:
  - negative number: return N lines from the end of buffer
  - `0`: do not return any line
  - positive number: return N lines from the beginning of buffer
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
        "id": 1709932823238637,
        "name": "core.weechat",
        "short_name": "weechat",
        "number": 1,
        "type": "formatted",
        "title": "WeeChat 4.2.0-dev (C) 2003-2023 - https://weechat.org/",
        "modes": "",
        "input_prompt": "",
        "input": "",
        "input_position": 0,
        "input_multiline": false,
        "nicklist": false,
        "nicklist_case_sensitive": false,
        "nicklist_display_groups": true,
        "local_variables": {
            "plugin": "core",
            "name": "weechat"
        },
        "keys": []
    },
    {
        "id": 1709932823423765,
        "name": "irc.server.libera",
        "short_name": "libera",
        "number": 2,
        "type": "formatted",
        "title": "IRC: irc.libera.chat/6697 (2001:4b7a:a008::6667)",
        "modes": "",
        "input_prompt": "",
        "input": "",
        "input_position": 0,
        "input_multiline": false,
        "nicklist": false,
        "nicklist_case_sensitive": false,
        "nicklist_display_groups": true,
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
        },
        "keys": []
    },
    {
        "id": 1709932823649069,
        "name": "irc.libera.#weechat",
        "short_name": "#weechat",
        "number": 3,
        "type": "formatted",
        "title": "Welcome to the WeeChat official support channel",
        "modes": "+nt",
        "input_prompt": "\u001b[92m@\u001b[96malice\u001b[48;5;22m(\u001b[39mi\u001b[48;5;22m)",
        "input": "",
        "input_position": 0,
        "input_multiline": false,
        "nicklist": true,
        "nicklist_case_sensitive": false,
        "nicklist_display_groups": false,
        "local_variables": {
            "plugin": "irc",
            "name": "libera.#weechat",
            "type": "channel",
            "server": "libera",
            "channel": "#weechat",
            "nick": "alice",
            "host": "~alice@example.com"
        },
        "keys": []
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
    "id": 1709932823238637,
    "name": "core.weechat",
    "short_name": "weechat",
    "number": 1,
    "type": "formatted",
    "title": "WeeChat 4.2.0-dev (C) 2003-2023 - https://weechat.org/",
    "modes": "",
    "input_prompt": "",
    "input": "",
    "input_position": 0,
    "input_multiline": false,
    "nicklist": false,
    "nicklist_case_sensitive": false,
    "nicklist_display_groups": true,
    "local_variables": {
        "plugin": "core",
        "name": "weechat"
    },
    "keys": [],
    "lines": [
        {
            "id": 10,
            "y": -1,
            "date": "2023-12-24T08:17:20.786538Z",
            "date_printed": "2023-12-24T08:17:20.786538Z",
            "displayed": true,
            "highlight": false,
            "notify_level": 0,
            "prefix": "",
            "message": "Plugins loaded: alias, buflist, charset, exec, fifo, fset, guile, irc, javascript, logger, lua, perl, php, python, relay, ruby, script, spell, tcl, trigger, typing, xfer",
            "tags": []
        }
    ]
}
```

Request example: get a channel buffers with nicks:

```bash
curl -L -u 'plain:secret_password' \
  'https://localhost:9000/api/buffers/irc.libera.%23weechat?nicks=true'
```

Response:

```text
HTTP/1.1 200 OK
```

```json
{
    "id": 1709932823649069,
    "name": "irc.libera.#weechat",
    "short_name": "#weechat",
    "number": 3,
    "type": "formatted",
    "title": "Welcome to the WeeChat official support channel",
    "modes": "+nt",
    "input_prompt": "\u001b[92m@\u001b[96malice\u001b[48;5;22m(\u001b[39mi\u001b[48;5;22m)",
    "input": "",
    "input_position": 0,
    "input_multiline": false,
    "nicklist": true,
    "nicklist_case_sensitive": false,
    "nicklist_display_groups": false,
    "local_variables": {
        "plugin": "irc",
        "name": "libera.#weechat",
        "type": "channel",
        "server": "libera",
        "channel": "#weechat",
        "nick": "alice",
        "host": "~alice@example.com"
    },
    "keys": [],
    "nicklist_root": {
        "id": 0,
        "parent_group_id": -1,
        "name": "root",
        "color_name": "",
        "color": "",
        "visible": false,
        "groups": [
            {
                "id": 1709932823649181,
                "parent_group_id": 0,
                "name": "000|o",
                "color_name": "weechat.color.nicklist_group",
                "color": "\u001b[32m",
                "visible": true,
                "groups": [],
                "nicks": [
                    {
                        "id": 1709932823649184,
                        "parent_group_id": 1709932823649181,
                        "prefix": "@",
                        "prefix_color_name": "lightgreen",
                        "prefix_color": "\u001b[92m",
                        "name": "alice",
                        "color_name": "bar_fg",
                        "color": "",
                        "visible": true
                    }
                ]
            },
            {
                "id": 1709932823649189,
                "parent_group_id": 0,
                "name": "001|h",
                "color_name": "weechat.color.nicklist_group",
                "color": "\u001b[32m",
                "visible": true,
                "groups": [],
                "nicks": []
            },
            {
                "id": 1709932823649203,
                "parent_group_id": 0,
                "name": "002|v",
                "color_name": "weechat.color.nicklist_group",
                "color": "\u001b[32m",
                "visible": true,
                "groups": [],
                "nicks": []
            },
            {
                "id": 1709932823649210,
                "parent_group_id": 0,
                "name": "999|...",
                "color_name": "weechat.color.nicklist_group",
                "color": "\u001b[32m",
                "visible": true,
                "groups": [],
                "nicks": []
            }
        ],
        "nicks": []
    }
}
```

Request example: get fset buffer with its local keys:

```bash
curl -L -u 'plain:secret_password' \
  'https://localhost:9000/api/buffers/fset.fset'
```

Response:

```text
HTTP/1.1 200 OK
```

```json
{
    "id": 1709932823897200,
    "name": "fset.fset",
    "short_name": "",
    "number": 4,
    "type": "free",
    "title": "\u001b[96m1/\u001b[36m3565 | Filter: \u001b[93m* | Sort: \u001b[97m~name | Key(input): alt+space=toggle boolean, alt+'-'(-)=subtract 1 or set, alt+'+'(+)=add 1 or append, alt+f,alt+r(r)=reset, alt+f,alt+u(u)=unset, alt+enter(s)=set, alt+f,alt+n(n)=set new value, alt+f,alt+a(a)=append, alt+','=mark/unmark, shift+down=mark and move down, shift+up=move up and mark, ($)=refresh, ($$)=unmark/refresh, (m)=mark matching options, (u)=unmark matching options, alt+p(p)=toggle plugins desc, alt+v(v)=toggle help bar, ctrl+x(x)=switch format, (q)=close buffer",
    "modes": "",
    "input_prompt": "",
    "input": "",
    "input_position": 0,
    "input_multiline": false,
    "nicklist": false,
    "nicklist_case_sensitive": false,
    "nicklist_display_groups": true,
    "local_variables": {
        "plugin": "fset",
        "name": "fset",
        "type": "option",
        "filter": "*"
    },
    "keys": [
        {
            "key": "ctrl-l",
            "command": "/fset -refresh"
        },
        {
            "key": "ctrl-n",
            "command": "/eval ${if:${weechat.bar.buflist.hidden}?/fset -down:/buffer +1}"
        },
        {
            "key": "ctrl-x",
            "command": "/fset -format"
        },
        {
            "key": "down",
            "command": "/fset -down"
        },
        {
            "key": "f11",
            "command": "/fset -left"
        },
        {
            "key": "f12",
            "command": "/fset -right"
        },
        {
            "key": "meta-+",
            "command": "/fset -add 1"
        },
        {
            "key": "meta--",
            "command": "/fset -add -1"
        },
        {
            "key": "meta-comma",
            "command": "/fset -mark"
        },
        {
            "key": "meta-end",
            "command": "/fset -go end"
        },
        {
            "key": "meta-f,meta-a",
            "command": "/fset -append"
        },
        {
            "key": "meta-f,meta-n",
            "command": "/fset -setnew"
        },
        {
            "key": "meta-f,meta-r",
            "command": "/fset -reset"
        },
        {
            "key": "meta-f,meta-u",
            "command": "/fset -unset"
        },
        {
            "key": "meta-home",
            "command": "/fset -go 0"
        },
        {
            "key": "meta-p",
            "command": "/mute /set fset.look.show_plugins_desc toggle"
        },
        {
            "key": "meta-q",
            "command": "/input insert meta-q fset ${property}"
        },
        {
            "key": "meta-return",
            "command": "/fset -set"
        },
        {
            "key": "meta-space",
            "command": "/fset -toggle"
        },
        {
            "key": "meta-v",
            "command": "/bar toggle fset"
        },
        {
            "key": "shift-down",
            "command": "/fset -mark; /fset -down"
        },
        {
            "key": "shift-up",
            "command": "/fset -up; /fset -mark"
        },
        {
            "key": "up",
            "command": "/fset -up"
        }
    ]
}
```

#### Sub-resource: buffers / lines

Return lines in a buffer.

Endpoints:

```http
GET /api/buffers/{buffer_id}/lines
GET /api/buffers/{buffer_id}/lines/{line_id}
GET /api/buffers/{buffer_name}/lines
GET /api/buffers/{buffer_name}/lines/{line_id}
```

Path parameters:

- `buffer_id` (integer, **required**): buffer unique identifier (not to be confused with the buffer number, which is different)
- `buffer_name` (string, **required**): buffer name
- `line_id` (integer, optional): return a single line with this identifier

Query parameters:

- `lines` (integer, optional, default: all lines): number of lines to return:
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
        "displayed": true,
        "highlight": false,
        "notify_level": 0,
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
        "displayed": true,
        "highlight": false,
        "notify_level": 0,
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
        "displayed": true,
        "highlight": false,
        "notify_level": 0,
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
GET /api/buffers/{buffer_id}/nicks
GET /api/buffers/{buffer_name}/nicks
```

Path parameters:

- `buffer_id` (integer, **required**): buffer unique identifier (not to be confused with the buffer number, which is different)
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
    "id": 0,
    "parent_group_id": -1,
    "name": "root",
    "color_name": "",
    "color": "",
    "visible": false,
    "groups": [
        {
            "id": 1709932823649181,
            "parent_group_id": 0,
            "name": "000|o",
            "color_name": "weechat.color.nicklist_group",
            "color": "\u001b[32m",
            "visible": true,
            "groups": [],
            "nicks": [
                {
                    "id": 1709932823649184,
                    "parent_group_id": 1709932823649181,
                    "prefix": "@",
                    "prefix_color_name": "lightgreen",
                    "prefix_color": "\u001b[92m",
                    "name": "alice",
                    "color_name": "bar_fg",
                    "color": "",
                    "visible": true
                }
            ]
        },
        {
            "id": 1709932823649189,
            "parent_group_id": 0,
            "name": "001|h",
            "color_name": "weechat.color.nicklist_group",
            "color": "\u001b[32m",
            "visible": true,
            "groups": [],
            "nicks": []
        },
        {
            "id": 1709932823649203,
            "parent_group_id": 0,
            "name": "002|v",
            "color_name": "weechat.color.nicklist_group",
            "color": "\u001b[32m",
            "visible": true,
            "groups": [],
            "nicks": []
        },
        {
            "id": 1709932823649210,
            "parent_group_id": 0,
            "name": "999|...",
            "color_name": "weechat.color.nicklist_group",
            "color": "\u001b[32m",
            "visible": true,
            "groups": [],
            "nicks": []
        }
    ],
    "nicks": []
}
```

### Resource: hotlist

Return hotlist.

Endpoints:

```http
GET /api/hotlist
```

Request example:

```bash
curl -L -u 'plain:secret_password' 'https://localhost:9000/api/hotlist'
```

Response:

```text
HTTP/1.1 200 OK
```

```json
[
    {
        "priority": 0,
        "date": "2024-03-17T16:38:51.572834Z",
        "buffer_id": 1710693531508204,
        "count": [
            44,
            0,
            0,
            0
        ]
    },
    {
        "priority": 0,
        "date": "2024-03-17T16:38:51.573028Z",
        "buffer_id": 1710693530395959,
        "count": [
            14,
            0,
            0,
            0
        ]
    },
    {
        "priority": 0,
        "date": "2024-03-17T16:38:51.611617Z",
        "buffer_id": 1710693531529248,
        "count": [
            4,
            0,
            0,
            0
        ]
    }
]
```

### Resource: input

Send command or text to a buffer.

Endpoint:

```http
POST /api/input
```

Body parameters:

- `buffer_id` (integer, optional): buffer unique identifier (not to be confused with the buffer number, which is different)
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
- `input` (boolean, optional, default: `true`): `true` to synchronize input from remote to local (used only if `sync` is `true`)
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

Authentication must be done only one time when a websocket is used (see [Authentication](#authentication)).

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

Note: the `Sec-WebSocket-Accept` value returned is the SHA-1 hash of the value received, concatenated to the GUID `258EAFA5-E914-47DA-95CA-C5AB0DC85B11` (the SHA-1 is encoded in base64). \
In the example above, the SHA-1 of `2XE8VAJktqi3Tpw5QnfxVQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11` is `PaY9vRflWeOKuD0/F7e5gD9At9U=` (in base64).

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
- `request` (string): the request (example: `GET /api/buffers?lines=-100`)
- `request_body` (object): the request body, or `null` if the request had no body

When the response has a body, these two extra fields are returned:

- `body_type` (string): type of objects returned in body (see below)
- `body` (object or array): the body returned

Body types that can be returned:

- `handshake`
- `version`
- `buffer`
- `line`
- `nick_group`
- `nick`
- `hotlist`
- `ping`

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
    "request": "GET /api/version",
    "request_body": null,
    "body_type": "version",
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
        "buffer_name": "irc.libera.#weechat",
        "command": "hello!"
    }
}
```

Response:

```json
{
    "code": 204,
    "message": "No Content",
    "request": "POST /api/input",
    "request_body": {
        "buffer_name": "irc.libera.#weechat",
        "command": "hello!"
    }
}
```

WeeChat pushes data to the client at any time on some events: when lines are displayed, buffers added/removed/changed, nicks added/removed/changed, etc.

The JSON sent has `code` set to `0`, `message` set to `Event` and an extra object `event` with the following data:

- `name` (string): the event name (name of signal or hsignal)
- `buffer_id` (integer): the buffer unique identifier, set only for sub-objects, -1 in other cases

The following events are sent to the client, according to synchronization options:

Event                     | Buffer id | Body type     | Body
------------------------- | --------- | ------------- | -------------------------------
`buffer_opened`           | buffer id | `buffer`      | buffer with all lines and nicks
`buffer_type_changed`     | buffer id | `buffer`      | buffer
`buffer_moved`            | buffer id | `buffer`      | buffer
`buffer_merged`           | buffer id | `buffer`      | buffer
`buffer_unmerged`         | buffer id | `buffer`      | buffer
`buffer_hidden`           | buffer id | `buffer`      | buffer
`buffer_unhidden`         | buffer id | `buffer`      | buffer
`buffer_renamed`          | buffer id | `buffer`      | buffer
`buffer_title_changed`    | buffer id | `buffer`      | buffer
`buffer_localvar_added`   | buffer id | `buffer`      | buffer
`buffer_localvar_changed` | buffer id | `buffer`      | buffer
`buffer_localvar_removed` | buffer id | `buffer`      | buffer
`buffer_cleared`          | buffer id | `buffer`      | buffer
`buffer_closing`          | buffer id | `buffer`      | buffer
`buffer_closed`           | buffer id | (not defined) | (not defined)
`buffer_line_added`       | buffer id | `line`        | buffer line
`input_text_changed`      | buffer id | `buffer`      | buffer
`input_text_cursor_moved` | buffer id | `buffer`      | buffer
`upgrade`                 | -1        | (not defined) | (not defined)
`upgrade_ended`           | -1        | (not defined) | (not defined)
`nicklist_group_changed`  | buffer id | `nick_group`  | nick group
`nicklist_group_added`    | buffer id | `nick_group`  | nick group
`nicklist_group_removing` | buffer id | `nick_group`  | nick group
`nicklist_nick_added`     | buffer id | `nick`        | nick
`nicklist_nick_removing`  | buffer id | `nick`        | nick
`nicklist_nick_changed`   | buffer id | `nick`        | nick

Note: the `upgrade` and `upgrade_ended` events are sent only if the client
is connected with plain text (no TLS), because with TLS the client is
disconnected before the upgrade is done (upgrade of TLS connections is not
supported).

Example: new buffer: channel `#weechat` has been joined:

```json
{
    "code": 0,
    "message": "Event",
    "event": {
        "name": "buffer_opened",
        "buffer_id": 1709932823649069
    },
    "body_type": "buffer",
    "body": {
        "id": 1709932823649069,
        "name": "irc.libera.#test",
        "short_name": "",
        "number": 4,
        "type": "formatted",
        "title": "",
        "modes": "+nt",
        "input_prompt": "\u001b[92m@\u001b[96malice\u001b[48;5;22m(\u001b[39mi\u001b[48;5;22m)",
        "input": "",
        "input_position": 0,
        "input_multiline": false,
        "nicklist": true,
        "nicklist_case_sensitive": false,
        "nicklist_display_groups": false,
        "local_variables": {
            "plugin": "irc",
            "name": "libera.#test",
            "type": "channel",
            "nick": "alice",
            "host": "~alice@example.com",
            "server": "libera",
            "channel": "#test"
        },
        "keys": [],
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
        "buffer_id": 1709932823649069
    },
    "body_type": "line",
    "body": {
        "id": 5,
        "index": -1,
        "date": "2024-01-07T08:54:00.179483Z",
        "date_printed": "2024-01-07T08:54:00.179483Z",
        "displayed": true,
        "highlight": false,
        "notify_level": 0,
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
        "buffer_id": 1709932823649069
    },
    "body_type": "nick",
    "body": {
        "id": 1709932823649902,
        "parent_group_id": 1709932823649181,
        "prefix": "@",
        "prefix_color_name": "lightgreen",
        "prefix_color": "\u001b[92m",
        "name": "bob",
        "color_name": "bar_fg",
        "color": "",
        "visible": true
    }
}
```

Example: channel buffer `#weechat` has been closed:

```json
{
    "code": 0,
    "message": "Event",
    "event": {
        "name": "buffer_closed",
        "buffer_id": 1709932823649069
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
        "buffer_id": -1
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
        "buffer_id": -1
    }
}
```

### Remote

A new `/remote` command is added to manage and connect to remotes. A remote is
another WeeChat running with a relay "api".

Each remote has these options:

Name            | Required | Description                          | Example
--------------- | -------- | ------------------------------------ | ------------------------
`autoconnect`   | **Yes**  | Auto-connect when WeeChat starts     | `on`
`password`      | **Yes**  | Password of remote relay             | `secret`
`proxy`         | No       | Name of proxy used to connect        | `my_proxy`
`tls_verify`    | **Yes**  | Verify TLS certificate (recommended) | `on`
`totp_secret`   | No       | TOTP secret of remote relay          | `secretbase32`
`url`           | **Yes**  | URL of remote                        | `https://localhost:9000`

#### Command /remote

```text
[relay]  /remote  list|listfull [<name>]
                  add <name> <url> [-<option>[=<value>]]
                  connect <name>
                  send <name> <json>
                  disconnect <name>
                  rename <name> <new_name>
                  del <name>

control of remote relay servers

      list: list remote relay servers (without argument, this list is displayed)
  listfull: list remote relay servers (verbose)
       add: add a remote relay server
      name: name of remote relay server, for internal and display use; this name is used to connect to the remote relay and to set remote relay options: relay.remote.name.xxx
       url: URL of the remote relay, format is https://example.com:9000 or http://example.com:9000 (plain-text connection, not recommended)
    option: set option for remote relay
   connect: connect to a remote relay server
      send: send JSON data to a remote relay server
disconnect: disconnect from a remote relay server
    rename: rename a remote relay server
       del: delete a remote relay server

Examples:
  /remote add example https://localhost:9000 -password=my_secret_password -totp_secret=secrettotp
  /remote connect example
  /remote del example
```

## Planning

The changes must be implemented in this order:

1. Add "api" protocol with JSON support and plain-text password authentication
2. Add support of websocket extension "permessage-deflate"
3. Add support of hashed passwords (SHA and PBKDF2)
4. Add "handshake" resource
5. Add command `/remote` to manage and connect to remote WeeChat relay/api
6. Add OpenAPI document with tests using [schemathesis](https://github.com/schemathesis/schemathesis)

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-005-relay-http-rest-api.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-005-relay-http-rest-api.md)
