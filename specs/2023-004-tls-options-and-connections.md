# TLS options and connections

- Author: [Sébastien Helleu](https://github.com/flashcode)
- License: CC BY-NC-SA 4.0
- Created on: 2023-04-04
- Last updated: 2023-04-14
- Issue: [#1903](https://github.com/weechat/weechat/issues/1903): rename "ssl" to "tls", connect to IRC servers with TLS and port 6697 by default
- Status: implemented
- Target WeeChat version: 4.0.0

## Context

WeeChat makes clear text connections by default (IRC, relay).
TLS options are named "ssl".

## Goals

Purpose of this specification is to rename `ssl` to `tls` everywhere: options, commands, documentation.\
Connection to IRC servers must be with TLS and port 6697 by default.

## Out of scope

N/A

## Changes

### Options

#### Core

The following options are renamed in core:

Option                          | New name
------------------------------- | -------------------------------
`weechat.color.status_name_ssl` | `weechat.color.status_name_tls`

The help on option `weechat.network.gnutls_ca_system` is updated to replace `SSL` by `TLS`.

#### IRC

The following options are renamed in IRC plugin:

Option                               | New name
------------------------------------ | ------------------------------------
`irc.server_default.ssl`             | `irc.server_default.tls`
`irc.server_default.ssl_cert`        | `irc.server_default.tls_cert`
`irc.server_default.ssl_dhkey_size`  | `irc.server_default.tls_dhkey_size`
`irc.server_default.ssl_fingerprint` | `irc.server_default.tls_fingerprint`
`irc.server_default.ssl_password`    | `irc.server_default.tls_password`
`irc.server_default.ssl_priorities`  | `irc.server_default.tls_priorities`
`irc.server_default.ssl_verify`      | `irc.server_default.tls_verify`
`irc.server.xxx.ssl`                 | `irc.server.xxx.tls`
`irc.server.xxx.ssl_cert`            | `irc.server.xxx.tls_cert`
`irc.server.xxx.ssl_dhkey_size`      | `irc.server.xxx.tls_dhkey_size`
`irc.server.xxx.ssl_fingerprint`     | `irc.server.xxx.tls_fingerprint`
`irc.server.xxx.ssl_password`        | `irc.server.xxx.tls_password`
`irc.server.xxx.ssl_priorities`      | `irc.server.xxx.tls_priorities`
`irc.server.xxx.ssl_verify`          | `irc.server.xxx.tls_verify`

The IRC configuration file `irc.conf` is bumped to version 2 and the options above are automatically renamed to the new name during upgrade of config, the user values are preserved.

#### Relay

The following options are renamed in relay plugin:

Option                         | New name
------------------------------ | ------------------------------
`relay.network.ssl_cert_key`   | `relay.network.tls_cert_key`
`relay.network.ssl_priorities` | `relay.network.tls_priorities`

The protocol `ssl` is renamed to `tls`.

The relay configuration file `relay.conf` is bumped to version 2 and the options above are automatically renamed to the new name during upgrade of config, the user values are preserved.

### Commands

The help on following commands is updated to replace `SSL` by `TLS`:

- `/upgrade`

The option `sslcertkey` in command `/relay` is renamed to `tlscertkey`.

### IRC server

The default value of option `irc.server_default.tls` becomes `on`.

The new servers created with commands `/server` and `/connect` are now using port `6697` and TLS enabled by default (if option `irc.server_default.tls` is set to `on`), following the [RFC 7194](https://www.rfc-editor.org/rfc/rfc7194).

Examples, supposing option `irc.server_default.tls` is set to `on`:

Command                                          | Port used | TLS enabled?
------------------------------------------------ | --------- | ------------
`/server add libera irc.libera.chat`             | 6697      | Yes
`/server add libera irc.libera.chat/7000`        | 7000      | Yes
`/server add libera irc.libera.chat/6667 -notls` | 6667      | No

If option `irc.server_default.tls` is set to `off` (which is the case after upgrade if the option has not been changed from default value), the behavior is the same as WeeChat ≤ 3.8:

Command                                        | Port used | TLS enabled?
---------------------------------------------- | --------- | ------------
`/server add libera irc.libera.chat`           | 6667      | No
`/server add libera irc.libera.chat/6666`      | 6666      | No
`/server add libera irc.libera.chat/7000 -tls` | 7000      | Yes

For convenience, extra information is displayed when a new server is added:

- all addresses with port (possibly assigned automatically by WeeChat)
- TLS: enabled/disabled

For example:

```text
/server add test irc1.example.org,irc2.example.org/7000

irc: server added: test -> irc1.example.org/6697, irc2.example.org/7000 (SSL: enabled)
```

In addition, internal variables of the IRC server are renamed:

Variable        | New name
--------------- | ---------------
`ssl_connected` | `tls_connected`

The following script is affected and must be updated:

- keepnick.py

## Planning

The changes must be implemented in this order:

1. Rename core options and update help on options
2. Update help of core commands
3. Rename IRC options (bump version of `irc.conf`)
4. Rename relay options (bump version of `relay.conf`) and protocol
5. Make TLS and port 6697 default in IRC plugin
6. Update script keepnick.py.

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-004-tls-options-and-connections.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-004-tls-options-and-connections.md)
